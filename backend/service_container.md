Система построена на трейтах. В контроллерах есть возможность переопределить метод getAutoWiredParameters (https://goo.su/FpWJ4D).

Базовый трейт должен быть добавлен через ключевое слово use во все дочерние трейты 
```php
<?php
namespace Polus\DI\ServiceProviders;

use Bitrix\Main\Localization\Loc;
use Exception;
use ReflectionClass;

Loc::loadMessages(__FILE__);

/**
 * Реализация системы сервис-контейнеров для внедрения зависимостей в методы контроллеров.
 * Каждый трейт-сервис должен включать в себя BaseService.
 */
trait BaseService {

    /**
     * Описывает связи между интерфейсом и реализацией, например:
     * IWriter::class => [
     *  'class' => XlsxWriter::class,
     *  'parameters' => ['test_parameter']
     * ]
     *
     * @return array
     */
    protected function getContainer(): array {
        return [];
    }

    /**
     * Через рефлексию создает реализацию переданного интерфейса.
     * Правила сопоставления должны быть описаны в @see BaseService::getContainer()
     *
     * @param string $interface
     * @return object|null
     * @throws \Bitrix\Main\ArgumentException
     * @throws \Bitrix\Main\ArgumentTypeException
     * @throws \Bitrix\Main\NotImplementedException
     * @throws \ReflectionException
     */
    protected function build(string $interface): ?object {
        $container = $this->getContainer();

        $buildData = $container[$interface] ?? null;

        if (is_null($buildData) || !$buildData['class'] || !class_exists($buildData['class'])) {
            throw new Exception(
                Loc::getMessage(
                    'POLUS_DI_SERVICE_CLASS_NOT_FOUND',
                    ['#CLASS#' => $interface]
                )
            );
        }

        $buildClass = $buildData['class'];

        $refClass = new ReflectionClass($buildClass);

        if (!$refClass->isSubclassOf($interface)) {
            throw new Exception(
                Loc::getMessage(
                    'POLUS_DI_SERVICE_IMPLEMENTS_ERROR',
                    ['#CLASS#' => $buildClass, '#INTERFACE' => $interface]
                )
            );
        }

        $parameters = $buildData['parameters'] ?? [];

        if (empty($parameters)) {
            return $refClass->newInstance();
        }

        foreach ($parameters as &$parameter) {
            if (in_array($parameter, array_keys($container))) {
                $parameter = $this->build($parameter);
            }
        }

        return $refClass->newInstanceArgs($parameters);
    }
}
```

Реализация сервис-контейнера для корзины

```php
namespace Polus\SiteCore\DI\ServiceProviders;

/**
 * Сервис-контейнер для корзины
 */
trait BasketService {
    use BaseService;

    /**
     * Возвращает зависимости
     *
     * @return array[]
     * @throws \Bitrix\Main\ArgumentException
     * @throws \Bitrix\Main\ArgumentTypeException
     * @throws \Bitrix\Main\NotImplementedException
     */
    protected function getContainer(): array {
        return [
            IWriter::class => [
                'class' => SpreadsheetXlsWriter::class,
                'parameters' => [Helper::getTmpFileName('basket_info')]
            ],
            IBasketExport::class => [
                'class' => BasketExport::class,
                'parameters' => [
                    IWriter::class,
                    Sale\Basket::loadItemsForFUser(
                        Sale\Fuser::getId(), Context::getCurrent()->getSite()
                    ),
                    BaseRepository::class
                ]
            ],
            BaseRepository::class => [
                'class' => IblockRepository::class,
                'parameters' => [Constants::IBLOCK_CATALOG_CODE]
            ]
        ];
    }

    /**
     * Возвращает массив с параметрами, которые необходимо внедрить
     *
     * @return DIParameter[]
     * @throws \Bitrix\Main\Engine\AutoWire\BinderArgumentException
     */
    public function getAutoWiredParameters(): array {
        return [
            new DIParameter(
                IBasketExport::class,
                'basketExport',
                function() {
                    return $this->build(IBasketExport::class);
                }
            )
        ];
    }
}
```

Пример использования в методе контроллера корзины

```php
/**
 * Возвращает файл с содержимым корзины
 *
 * @param IBasketExport $basketExport
 * @return array|false
 */
public function getBasketFileAction(IBasketExport $basketExport) {

    try {

        $basketExport->export();

        return [
            "src" => $basketExport->getFileName()
        ];

    } catch (Exception $ex) {
        $this->addError(new Error($ex->getMessage()));
    }

    return false;
}
```