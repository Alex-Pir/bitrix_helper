
У пользователя есть товары, которые он чаще всего покупает, их идентификаторы лежат в отдельной таблице.
Задача состоит в том, чтобы выводить эти товары в начале списка, а остальные товары сортировать по названию.

```php
global $USER;

$abcProducts = AbcTable::getList([
    "filter" => ["USER_ID" => $USER->GetID()],
    "select" => ["UF_PRODUCT_ID"]
])->fetchAll();

$ids = [];

foreach ($abcProducts as $product) {
$ids[] = $product["UF_PRODUCT_ID"];
}

$class = Bitrix\Iblock\Iblock::wakeUp(2)->getEntityDataClass();

/**
* union не позволяет сортировать подзапрос, поэтому нужен способ, чтобы при сортировке
 * результаты основного запроса не перемешивались с дополнительным.
 * 
 * Добавляем вычисляемое поле "GROUP" и делаем так, чтобы в основном запросе оно обозначалось как 1,
 * а в дополнительном как 2. Это позволит нам при сортировке сначала использовать "GROUP", что уже не
 * даст смешиваться результатам. В качестве второй сортировки возьмем "NAME". Первая группа тоже
 * отсортируется по "NAME", но они хотя бы будут отдельно друг от друга.
 */
$result = $class::query()
    ->addSelect("ID")
    ->addSelect("NAME")
    ->addSelect("GROUP")
    ->addFilter("ID", $ids)
    ->addFilter("ACTIVE", "Y")
    ->registerRuntimeField("GROUP", [
        "expression" => ['%s + 1', 'ACTIVE']
    ])
    ->union($class::query()
         ->addFilter("ACTIVE", "Y")
        ->addSelect("ID")
        ->addSelect("NAME")
        ->addSelect("GROUP")
        ->registerRuntimeField("GROUP", [
            "expression" => ['%s + 2', 'ACTIVE']
        ])
    )
    ->setUnionLimit(20)
    ->setUnionOrder(["GROUP" => "ASC", "NAME" => "ASC"])
    ->exec();
```