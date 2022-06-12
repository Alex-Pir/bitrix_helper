Фасад к битриксовой системе кеширования

```php
<?php
namespace Polus\Core;

use CPHPCache;
use Exception;

/**
 * Класс предназначен для работы с кешем
 *
 * Class Cache
 * @package Polus\Core
 */
class Cache
{
    /** @var int время кеширования на длительный срок */
    const CACHE_TIME_LONG = 36000000;

    /** @var int время кеширования на короткий срок */
    const CACHE_TIME_SHORT = 3600;

    /**
     * Сохранение данных в кеше
     *
     * @param string $cacheId
     * @param int $cacheTime
     * @param string $cacheDir
     * @param callable $function
     * @param $tag
     * @return false|mixed
     */
    public static function keep(string $cacheId, int $cacheTime, string $cacheDir, callable $function, $tag = null) {
        $obCache = new CPHPCache();

        if ($obCache->InitCache($cacheTime, $cacheId, $cacheDir)) {
            $result = $obCache->GetVars();
        } else {
            $obCache->StartDataCache();

            try {
                $result = call_user_func($function);
            } catch (Exception $ex) {
                AddMessage2Log($ex->getMessage());
                $obCache->AbortDataCache();
                return null;
            }

            if (defined("BX_COMP_MANAGED_CACHE") && $tag) {
                $GLOBALS["CACHE_MANAGER"]->StartTagCache($cacheDir);

                if (is_array($tag)) {
                    foreach ($tag as $value) {
                        $GLOBALS["CACHE_MANAGER"]->RegisterTag($value);
                    }
                } else {
                    $GLOBALS["CACHE_MANAGER"]->RegisterTag($tag);
                }

                $GLOBALS["CACHE_MANAGER"]->EndTagCache();
            }
            $obCache->EndDataCache($result);
        }

        return $result;
    }

    /**
     * Сохранение данных в кеше на длительный срок
     *
     * @param string $cacheId
     * @param string $cacheDir
     * @param callable $function
     * @param string|array $tag
     * @return false|mixed
     */
    public static function keepAlways(string $cacheId, string $cacheDir, callable $function, $tag = "") {
        return static::keep($cacheId, 99999999 * 60, $cacheDir, $function, $tag);
    }

    /**
     * Очистка кеша по тегу
     *
     * @param string $tag
     */
    public static function clearCacheByTag(string $tag) {
        if (defined("BX_COMP_MANAGED_CACHE") && trim($tag) !== '') {
            $GLOBALS["CACHE_MANAGER"]->ClearByTag($tag);
        }
    }
}
```

Пример использования:

```php
Cache::keep(
    md5(serialize([$parameter1, $parameter2])),
    Cache::CACHE_TIME_SHORT,
    '/web/page_directory/',
    function () use ($parameter1, $parameter2) {
        
    },
    "iblock_id_$iblockId"
);

Cache::keepAlways(
    md5(serialize([$parameter1, $parameter2])),
    '/web/page_directory/',
    function () use ($parameter1, $parameter2) {
        
    },
    "iblock_id_$iblockId"
);
```
