# Резюме

Добавляем топик `kitchenmotivation.placementreport.v1`, в который публикуем результаты подсчета расстановки и загруженности кухни в `KitchenMotivation`

Результаты подсчетов в дальнейшем будут выгружаться в data-платформу для аналитики.

# Мотивация, задача

Добавление топика `kitchenmotivation.placementreport.v1` в Kafka позволит передавать сообщения в data-платформу.

# Описание

Запуск cronjob с подсчетом и созданием событий будет запускаться каждый час сутки и создавать выгрузку для пиццерий подходящих по часовому поясу за -24 часа.

# dotNET contract

```C#
/// <summary>
/// Период отчета мощности пиццерии
/// </summary>
/// <param name="EventId">Идентификатор события</param>
/// <param name="CountryId">Идентификатор страны</param>
/// <param name="BusinessId">Идентификатор бизнеса</param>
/// <param name="UnitId">Идентификатор пиццерии</param>
/// <param name="AtUtc">Время события</param>
/// <param name="Begin">Начало периода</param>
/// <param name="End">Конец периода</param>
/// <param name="BaseWorkload">Загруженность станции раскатки</param>
/// <param name="ToppingsWorkload">Загруженность станции начинения</param>
/// <param name="SnackWorkload">Загруженность станции закусок</param>
/// <param name="PackingWorkload">Загруженность станции упаковки</param>
/// <param name="AverageDelay">Среднее время отклонения в приготовлении</param>
/// <param name="NotEnoughWorkers">Индикатор Недостаточно сотрудников</param>
/// <param name="TooMuchWorkers">Индикатор Лишние сотрудники</param>
/// <param name="WrongArrangement">Индикатор Несоответствие расстановки</param>
public record PlacementReport(
    Uuid EventId,
    int CountryId,
    Uuid BusinessId,
    Uuid UnitId,
    DateTime AtUtc,
    DateTime BeginUtc,
    DateTime EndUtc,
    int BaseWorkload,
    int ToppingsWorkload,
    int SnackWorkload,
    int PackingWorkload,
    TimeSpan AverageDelay,
    TimeSpan NotEnoughWorkers,
    TimeSpan TooMuchWorkers,
    TimeSpan WrongArrangement
    );
```


# JSON-scheme

```JSON
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "PlacementReport",
  "description": "Период отчета мощности пиццерии",
  "type": "object",
  "properties": {
  "EventId": {
    "description": "Идентификатор события",
    "type": "string",
    "format": "uuid"
  },
  "CountryId": {
    "description": "Идентификатор страны",
    "type": "integer"
  },
  "BusinessId": {
    "description": "Идентификатор бизнеса",
    "type": "string",
    "format": "uuid"
  },
  "UnitId": {
    "description": "Идентификатор пиццерии",
    "type": "string",
    "format": "uuid"
  },
  "AtUtc": {
    "description": "Время события",
    "type": "string",
    "format": "date-time"
  },
  "Begin": {
    "description": "Начало периода",
    "type": "string",
    "format": "date-time"
  },
  "End": {
    "description": "Конец периода",
    "type": "string",
    "format": "date-time"
  },
  "BaseWorkload": {
    "description": "Загруженность станции раскатки",
    "type": "integer"
  },
  "ToppingsWorkload": {
    "description": "Загруженность станции начинения",
    "type": "integer"
  },
  "SnackWorkload": {
    "description": "Загруженность станции закусок",
    "type": "integer"
  },
  "PackingWorkload": {
    "description": "Загруженность станции упаковки",
    "type": "integer"
  },
  "AverageDelay": {
    "description": "Среднее время отклонения в приготовлении",
    "type": "string",
    "format": "duration"
  },
  "NotEnoughWorkers": {
    "description": "Индикатор Недостаточно сотрудников",
    "type": "string",
    "format": "duration"
  },
  "TooMuchWorkers": {
    "description": "Индикатор Лишние сотрудники",
    "type": "string",
    "format": "duration"
  },
  "WrongArrangement": {
    "description": "Индикатор Несоответствие расстановки",
    "type": "string",
    "format": "duration"
  }
  },
  "required": [
    "EventId",
    "CountryId",
    "BusinessId",
    "UnitId",
    "AtUtc",
    "Begin",
    "End",
    "BaseWorkload",
    "ToppingsWorkload",
    "SnackWorkload",
    "PackingWorkload",
    "AverageDelay",
    "NotEnoughWorkers",
    "TooMuchWorkers",
    "WrongArrangement"
  ]
}
```


#  JSON example

```JSON
{
	"EventId": "0196f1377cb67908a897617a41baaaa3",
	"CountryId": 643,
	"BusinessId": "63d4829611ea45c8ae71394860a2481c",
	"UnitId": "000d3a240c719a8711e68aba13f7fe13",
	"AtUtc": "2025-05-21T05:00:57.089179Z",
    "BeginUtc": "2025-05-20T02:15:57.089179Z",
    "EndUtc": "2025-05-20T02:30:57.089179Z",
    "BaseWorkload": 15,
    "ToppingsWorkload": 25,
    "SnackWorkload": 20,
    "PackingWorkload": 0,
    "AverageDelay": "00:00:15",
    "NotEnoughWorkers": "00:01:03",
    "TooMuchWorkers": "00:03:01",
    "WrongArrangement": "00:00:00"
}
```


# Недостатки

Создание ещё одного топика в Kafka

# Обоснование и альтернативы

Альтернативы:

1) API - увеличит нагрузку на сервис и не позволит аналитикам строить графики на больших данных.

2) Создание событий в момент расчета, а не после - увеличит количество хранимых не используемых данных + отсутствие возможности подсчета параметров загруженности в "моменте", а не за прошедший период

