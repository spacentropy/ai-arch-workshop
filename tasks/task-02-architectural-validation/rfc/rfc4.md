Публикация событий в **kafka из монолита** при добавлении продуктов в стоп или удалении из стопа.
# Мотивация, задача
Необходимо отслеживать наличие продукта в стопе в конкретном ресторане, чтобы избежать ситуации, когда 
- При коммуникации с клиентом мы предложим ему продукт, который невозможно заказать в данный момент.
- *Какой-то ещё важный кейс*
Чтобы сервис, внешний по отношению к монолиту, мог асинхронно через прослушивание событий kafka синхронизировать в свою БД продукты, находящиеся в стопе, необходимо при установке продукта в стоп и выводе из стопа публиковать событие из монолита, содержащее данные о стопах продуктов.
# Описание
В ходе реализации необходимо добавить публикацию событий в Kafka в следующие места монолита:
- Публиковать событие в методе StopListService.Stop(). Если в стоп добавляется ингредиент, то так же в стоп будут добавлены все продукты, содержащие этот ингредиент.  В этом методе продукты массово добавляются в стоп, поэтому событий может быть несколько.
- Публиковать событие (события) в методе StopListService.StopOven(). В этом методе продукты массово добавляются в стоп, поэтому событий может быть несколько.
- Публиковать события в методе StopListService.RemoveStop(). Тут сложнее всего, т.к. тут не фигурируют конкретные идентификаторы продуктов. Чтобы их получить, нужно по ItemId поискать их в таблице stoppedproduct. Таким образом, мы добавим дополнительный запрос в БД.
Стоит учесть, что продукт может войти в стоп по разным причинам (ингредиент, печь), и при устранении одной из причин другая причина всё ещё может действовать. Например, начали чистить печь, и пицца "Пепперони" была добавлена в стоп. Через какое-то время, но до окончания чистки печи закончилась колбаса, и "Пепперони" была также добавлена в стоп по этой причине. После окончания чистки печи стоп на "Пепперони" по причине чистки будет отменен, но всё ещё будет действовать по причине окончания ингредиента. Поэтому, нужно также отслеживать конкретные причины (идентификаторы) добавления в стоп. 
*Технические подробности действий со стопами в монолите раскрыты в конце документа.*
## Событие для публикации в Kafka
Состав полей события может выглядеть следующим образом:
|Поле|Тип данных|Описание|
|-|-|-|
|EventId|string/Uuid|Идентификатор события, используется для дедупликации.|
|CountryId|Int|Код страны|
|MonolithId|string/Uuid|Uuid из Dodo.Tenants|
|AtUtc|datetime|Время события UTC|
|At|datetime|Время события|
|ProductId|string/Uuid|Идентификатор продукта, добавленного или убранного из стопа|
|ActionType|int|Тип действия - добавлен или убран из стопа
0 - AddedToStop
1 - RemovedFromStop|
|UnitId|string/Uuid|Идентификатор ресторана|
|StopId|string/Uuid|Идентификатор стопа. Продукт может войти в стоп по разным причинам (ингредиент, печь), и при устранении одной из причин другая причина всё ещё может действовать.|
|Reason|int|Причина стопа
0 - Добавлен вручную (продукт)
1 - Добавлен вручную (ингридиент)
2 - Обслуживание оборудования (чистка, поломка)
3* - Добавлен автоматически (остатки продуктов)
4* - Добавлен автоматически (остатки ингридиентов)
*Автоматического добавления пока нет, просто как пример на будущее|

```JSON
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ProductStopListEvent",
  "type": "object",
  "properties": {
    "EventId": {
      "type": "string",
      "description": "Unique identifier for the event"
    },
    "CountryId": {
      "type": "integer",
      "description": "ID representing the country"
    },
    "MonolithId": {
      "type": "uuid",
      "description": "ID for the monolithic system"
    },
    "AtUtc": {
      "type": "string",
      "format": "date-time",
      "description": "UTC date and time when the event occurred"
    },
    "At": {
      "type": "string",
      "format": "date-time",
      "description": "Local date and time when the event occurred"
    },
    "UnitId": {
      "type": "uuid",
      "description": "Unique identifier of the unit"
    },
    "StopId": {
      "type": "uuid",
      "description": "Unique identifier of the stop"
    },
    "ProductId": {
      "type": "uuid",
      "description": "Unique identifier of the product"
    },
    "ActionType": {
      "type": "int",
      "description": "Type of the action"
    },
    "Reason": {
      "type": "int",
      "description": "Reason of the stop"
    }
  },
  "required": [
    "EventId",
    "CountryId",
    "MonolithId",
    "AtUtc",
    "At",
    "ProductId",
    "ActionType",
    "UnitId",
    "StopId",
    "Reason"
  ]
}
```
# Технические подробности добавления в стоп
Продукт добавляется в стоп при следующих действиях:
- Добавление продукта в стоп
- Добавление ингредиента в стоп (при этом, в стоп добавляются все продукты, у которых есть этот ингредиент)
- Старт очистки печи (в стоп добавляются все продукты, выпекаемые в печи)
При обратных действиях (снятие ингредиента из стопа, окончание очистки печи) все затронутые продукты возвращаются из стопа.
## Добавление продуктов или ингредиентов в стоп
При установке продукта или ингредиента в стоп добавляются записи в две таблицы, например `stoppedItem` и `stoppedproduct`. Таблица `stoppeditem`- это как раз идентификатор действия установки в стоп. Когда добавляем в стоп ингредиент, то также добавляем в стоп все продукты с этим ингредиентом, и записи получают одинаковый ItemId.
```C#
public void Stop(int id, UUId uuid, StopType type, int stopReasonId, UserInfo stoppedBy)
		{
			var stoppedAt = stoppedBy.Department.CurrentDateTime;
			var productOrIngredientStopReason = (ProductOrIngredientStopReason) stopReasonId;

			if (IsProductStop(type))
			{
				StopProduct(id, uuid, stopReasonId, productOrIngredientStopReason, stoppedBy, stoppedAt);
			}
			else if (IsIngredientStop(type))
            {
                if (uuid == null) //todo delete it after the frontend sends uuid 
                    StopIngredient(id, stopReasonId, productOrIngredientStopReason, stoppedBy, stoppedAt);
                else StopIngredient(uuid, stopReasonId, productOrIngredientStopReason, stoppedBy, stoppedAt);
            }
		}
```
При удалении стопов (как отдельного продукта так и когда заканчиваем чистить печь) вызывается один и тот же эндпоинт `/Managment/StopList/RemoveFromStopList` и метод void `RemoveStop(UUId stoppedItemId, UserInfo stopRemovedBy)`
```C#
public void RemoveStop(UUId stoppedItemId, UserInfo stopRemovedBy)
{
	if (!CanRemoveItem(stoppedItemId, stopRemovedBy))
	{
		throw new StopItemException(Core.Resources.StopList.Text.Error_CanNotRemoveItem);
	}

	var stopRemovedAt = stopRemovedBy.Department.CurrentDateTime;

	_stopListRepository.RemoveStopItem(stoppedItemId, stopRemovedAt, stopRemovedBy.UserUUId);
}
```
## Очистка печи
Когда начинаем чистку, то вызываем метод `/Managment/StopList/AddProductsForOvenIntoStopList`
При чистке печи в стоп попадают все продукты, выпекаемые в печи. Всем продуктам, выпекаемым в печи, назначается ItemId.

```C#
public bool StopOven(UnitInfo unit, string reason, UserInfo startedBy)
		{
			if (_stopListRepository.IsOvenCleaningStarted(unit.Id))
			{
				throw new StopItemException(Text.OvenCleaningAlreadyStarted);
			}

			var stoppedAt = startedBy.Department.CurrentDateTime;
			var unitBakingProducts = _stopListRepository.GetBakingProducts();
			var productsInMenu = _productMenuService.GetProductsInUnitMenu(MenuDevice.Mobile, unit.Id);
			var productsToStop = unitBakingProducts.Intersect(productsInMenu)
                .Select(x => new StoppedDependentProduct(x.ProductId, x.ProductUUId))
                .ToList();

			var stopAuditInfo = new StopAuditInfo(stoppedAt, startedBy.UserUUId);
			var ovenCleaning = new OvenCleaning(startedBy.Unit.Id, startedBy.Unit.UUId, productsToStop, reason, stopAuditInfo);

			_stopListRepository.StopOven(ovenCleaning);

			return productsToStop.Any();
		}
```
Когда закончили чистку вызываем метод `/Managment/StopList/RemoveFromStopList` и в итоге выполняется такой SQL, где в том числе продукт удаляется из стопа по ItemId (идентификатору стопа):

```SQL
UPDATE stoplog s 
SET s.RemovedStopDateTime = @stopRemovedAt,
	s.RemovedStopUserId = 0,
	s.RemovedStopUserUUId = @removedByUserUUId
WHERE s.StoppedItemUUId = @stoppedItemUUId;

DELETE FROM stoppeddeliverysector WHERE StoppedItemUUId = @stoppedItemUUId;
DELETE FROM stoppeddeliveryunit WHERE StoppedItemUUId = @stoppedItemUUId;
DELETE FROM stoppedmaterialtype WHERE StoppedItemUUId = @stoppedItemUUId;
DELETE FROM stoppedordertypes WHERE StoppedItemUUId = @stoppedItemUUId;
DELETE FROM stoppedproduct WHERE StoppedItemUUId = @stoppedItemUUId;
DELETE FROM stoppedzonemainunit WHERE StoppedItemUUId = @stoppedItemUUId;
DELETE FROM stopsaleinfo WHERE StoppedItemUUId = @stoppedItemUUId;
DELETE FROM stoppeddelayedorders WHERE StoppedItemUUId = @stoppedItemUUId;
DELETE FROM stoppedItem WHERE UUId = @stoppedItemUUId;
```
Этот момент неявный, т.к. непосредственно в коде C# идентификаторы продуктов не фигуруют, и их нужно будет получить, чтобы добавить в ивент.
# Недостатки
*Почему **это** не делать? Что мы ухудшим?*
# Обоснование и альтернативы
Альтернативным вариантом может быть обращение к Legacy Facade для получения стопов в пиццерии. Однако в рассматриваемых сценариях (массовые сценарии "Брошеная корзина", массовые рассылки пуш-уведомлений, когда нужно проверить не находится ли в стопе продукты, о которых выполняется напоминание) потенциально возможен очень большой поток запросов к LF, даже если применить кеширование на несколько минут. В таких случаях предпочтительно иметь данные о стопах в собственной БД сервиса.
Также, существует топик в кафке `monolith.production.stoplog`, в который Debezium публикует изменения из таблицы stoplog (куда попадают все стопы), но этот источник данных применить нельзя, т.к. он будет отключен.
# История и предшественники
# Вопросы на обсуждение
## Публиковать один ивент с массивом продуктов или много ивентов на каждый продукт?
Изначально предполагалось публиковать один ивент, но в ходе обсуждения было решено, что лучше один ивент на каждый продукт, добавляемый в стоп.

