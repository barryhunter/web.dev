---
layout: post
title: Отслеживание общего использования памяти веб-страницей с помощью функции measureUserAgentSpecificMemory()
subhead: Узнайте, как измерять использование памяти веб-страницей в рабочей среде, чтобы обнаруживать регрессии.
description: Узнайте, как измерять использование памяти веб-страницей в рабочей среде, чтобы обнаруживать регрессии.
updated: 2020-10-19
date: 2020-04-13
authors:
  - ulan
hero: image/admin/Ne2U4cUtHG6bJ0YeIkt5.jpg
alt: Зеленый модуль ОЗУ. Фото Харрисона Бродбента (Harrison Broadbent) с Unsplash.
origin_trial:
  url: "https://developers.chrome.com/origintrials/#/view_trial/1281274093986906113"
tags:
  - blog
  - memory
  - javascript
feedback:
  - api
---

{% Aside 'caution' %}

**Обновления**

**23 апреля 2021 г.**: обновлен статус, уточнена область применения API — добавлено примечание о внепроцессных элементах iframe.

**20 января 2021 г.**: API `performance.measureMemory` переименован и теперь называется `performance.measureUserAgentSpecificMemory`. В Chrome 89 он по умолчанию доступен для [изолированных веб-страниц с разными источниками](/coop-coep). Кроме того, немного [изменился](https://github.com/WICG/performance-measure-memory/blob/master/ORIGIN_TRIAL.md#result-differences) формат результата по сравнению с версией Origin Trials. {% endAside %}

Браузеры автоматически управляют памятью веб-страниц. Когда веб-страница создает какой-либо объект, браузер выделяет фрагмент памяти для хранения этого объекта. Так как память — ограниченный ресурс, браузер выполняет сборку мусора. Это позволяет определить, что тот или иной объект больше не нужен, и можно освободить занимаемый им фрагмент памяти. Тем не менее функция обнаружения работает неидеально, и [было доказано](https://en.wikipedia.org/wiki/Halting_problem), что идеальное обнаружение — невыполнимая задача. Поэтому браузеры аппроксимируют понятие «объект необходим» понятием «объект достижим». Если веб-странице не удается обратиться к объекту через его переменные и поля других достижимых объектов, браузер может безопасно «утилизировать» объект. Разница между этими двумя понятиями приводит к утечкам памяти, как показано в приведенном ниже примере.

```javascript
const object = {a: new Array(1000), b: new Array(2000)};
setInterval(() => console.log(object.a), 1000);
```

Здесь больший массив `b` уже не нужен, но браузер не «утилизирует» его, так как он все еще доступен через свойство `object.b` в обратном вызове. Таким образом, возникает утечка памяти большего массива.

Утечки памяти — [обычное дело в Интернете](https://docs.google.com/presentation/d/14uV5jrJ0aPs0Hd0Ehu3JPV8IBGc3U8gU6daLAqj6NrM/edit#slide=id.p). Утечку памяти легко создать. Для этого достаточно забыть отменить регистрацию прослушивателя событий, случайно записать объекты из элемента iframe, не закрыть рабочий процесс, накапливать объекты в массивах и т. д. Если у веб-страницы имеется утечка памяти, использование памяти этой веб-страницей со временем растет, и веб-страница кажется пользователю медленной и «раздутой».

Первый шаг в решении этой проблемы — измерение. С помощью нового [API `performance.measureUserAgentSpecificMemory()`](https://github.com/WICG/performance-measure-memory) разработчики могут измерять использование памяти веб-страницами в рабочей среде и, соответственно, обнаруживать утечки памяти, не обнаруженные при тестировании в локальной среде.

## Отличия `performance.measureUserAgentSpecificMemory()` от старого API `performance.memory` {: #legacy-api }

Если вы знакомы с существующим нестандартным API `performance.memory`, вам, возможно, интересно, чем новый API отличается от него. Основное отличие состоит в том, что старый API возвращает размер кучи JavaScript, а новый оценивает объем памяти, используемый веб-страницей. Это различие важно, когда Chrome использует одну и ту же кучу для нескольких веб-страниц (или нескольких экземпляров одной и той же веб-страницы). В таких случаях результат старого API может быть случайным образом смещен. Так как старый API определяется терминами, зависящими от реализации, например «куча», попытки стандартизировать его обречены на провал.

Еще одно отличие состоит в том, что новый API выполняет измерение памяти во время сборки мусора. Это уменьшает «шум» в результатах, но для получения результатов может потребоваться некоторое время. Обратите внимание, что разработчики других браузеров могут реализовать новый API, не используя сборку мусора.

## Предлагаемые сценарии использования {: #use-cases }

То, каким образом веб-страница использует память, зависит от временных параметров событий, действий пользователя и операций сборки мусора. Вот почему API измерения памяти предназначен для агрегирования данных об использовании памяти в рабочей среде. Результаты индивидуальных вызовов менее полезны. Ниже приведены возможные сценарии использования такого API.

- Обнаружение регрессий во время развертывания новой версии веб-страницы, что позволяет обнаруживать новые утечки памяти.
- A/B-тестирование новой функции для оценки ее влияния на память и обнаружения утечек памяти.
- Корреляция использования памяти с продолжительностью сеанса для проверки наличия или отсутствия утечек памяти.
- Корреляция использования памяти с пользовательскими метриками, что позволяет понять, какое общее влияние оказывает использование памяти.

## Совместимость с браузерами {: #compatibility }

В настоящее время этот API поддерживается только в Chrome 83 в виде версии Origin Trials. Результаты работы API сильно зависят от его реализации, так как в браузерах применяются разные способы представления объектов в памяти и разные способы оценки использования памяти. Браузеры могут не учитывать некоторые области памяти, если надлежащий учет обходится слишком дорого либо невозможен. Таким образом, нельзя сравнивать результаты, получаемые в разных браузерах. Имеет смысл сравнивать только результаты для одного и того же браузера.

## Текущий статус {: #status }

<div><table>
    <thead>
      <tr>
        <th>Этап</th>
        <th>Статус</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>1. Создание эксплейнера</td>
        <td><a href="https://github.com/WICG/performance-measure-memory">Выполнен</a></td>
      </tr>
      <tr>
        <td>2. Создание первоначального проекта спецификации</td>
        <td><a href="https://wicg.github.io/performance-measure-memory/">Выполнен</a></td>
      </tr>
      <tr>
        <td>3. Сбор отзывов и доработка дизайна</td>
        <td><a href="#feedback">Выполняется</a></td>
      </tr>
      <tr>
        <td>4. Версия Origin Trials</td>
        <td><a href="https://developers.chrome.com/origintrials/#/view_trial/1281274093986906113">Выполнен</a></td>
      </tr>
      <tr>
        <td>5. Запуск</td>
        <td>Включено по умолчанию в Chrome 89</td>
      </tr>
    </tbody>
</table></div>

## Использование функции `performance.measureUserAgentSpecificMemory()` {: use }

### Включение на странице about://flags

Чтобы поэкспериментировать с функцией `performance.measureUserAgentSpecificMemory()` без токена Origin Trials, включите флаг `#experimental-web-platform-features` на странице `about://flags`.

### Обнаружение функций

Если среда выполнения не соответствует требованиям безопасности в части предотвращения утечки информации между источниками, работа функции `performance.measureUserAgentSpecificMemory()` может завершиться ошибкой [SecurityError](https://developer.mozilla.org/docs/Web/API/DOMException#exception-SecurityError). На этапе Origin Trials в Chrome для API требуется, чтобы была включена функция [Site Isolation](https://developers.google.com/web/updates/2018/07/site-isolation) (Изоляция сайтов). В окончательной версии API будет использоваться функция [изоляции между источниками](https://developer.mozilla.org/docs/Web/API/WindowOrWorkerGlobalScope/crossOriginIsolated). Веб-страница может включить функцию изоляции между источниками, настроив [заголовки COOP и COEP](https://docs.google.com/document/d/1zDlfvfTJ_9e8Jdc8ehuV4zMEu9ySMCiTGMS9y0GU92k/edit).

```javascript
if (performance.measureUserAgentSpecificMemory) {
  let result;
  try {
    result = await performance.measureUserAgentSpecificMemory();
  } catch (error) {
    if (error instanceof DOMException && error.name === 'SecurityError') {
      console.log('Этот контекст небезопасен.');
    } else {
      throw error;
    }
  }
  console.log(result);
}
```

### Тестирование в локальной среде

Chrome выполняет измерение памяти во время сборки мусора. Это означает, что API не сопоставляет обещание результата немедленно, а вместо этого ожидает следующей сборки мусора. API принудительно выполняет сборку мусора по истечении некоторого времени ожидания, которое на сегодняшний день равно 20 секундам. Запустив Chrome с флагом командной строки `--enable-blink-features='ForceEagerMeasureMemory'`, можно уменьшить время ожидания до нуля, что удобно при отладке и тестировании в локальной среде.

## Пример

Рекомендуемое использование этого API — создание глобального средства мониторинга памяти, которое делает выборки данных об использовании памяти всей веб-страницей и отправляет результаты на сервер для агрегирования и анализа. Самый простой способ — периодически (например, каждые `M` минут) делать выборку. Однако это приводит к появлению ошибок оценки, так как между выборками могут возникать пиковые показатели использования памяти. В приведенном ниже примере показано, как выполнять измерения памяти без ошибок с использованием [пуассоновского процесса](https://en.wikipedia.org/wiki/Poisson_point_process). Это гарантирует, что вероятность создания выборок будет одинакова для любых моментов времени ([демонстрация](https://performance-measure-memory.glitch.me/), [источник](https://glitch.com/edit/#!/performance-measure-memory?path=script.js:1:0)).

Прежде всего определите функцию, которая планирует следующее измерение памяти, используя функцию `setTimeout()` со случайными интервалами. Функцию следует вызывать после загрузки страницы в главном окне.

```javascript
function scheduleMeasurement() {
  if (!performance.measureUserAgentSpecificMemory) {
    console.log(
      'Функция performance.measureUserAgentSpecificMemory() недоступна.',
    );
    return;
  }
  const interval = measurementInterval();
  console.log(
    'Scheduling memory measurement in ' +
      Math.round(interval / 1000) +
      ' с.',
  );
  setTimeout(performMeasurement, interval);
}

// Запуск измерений после загрузки страницы в главном окне.
window.onload = function () {
  scheduleMeasurement();
};
```

Функция `measurementInterval()` вычисляет случайный интервал в миллисекундах, так что в среднем одно измерение выполняется каждые пять минут. Математические принципы, лежащие в основе этой функции, изложены в разделе «[Экспоненциальное распределение](https://en.wikipedia.org/wiki/Exponential_distribution#Computational_methods)».

```javascript
function measurementInterval() {
  const MEAN_INTERVAL_IN_MS = 5 * 60 * 1000;
  return -Math.log(Math.random()) * MEAN_INTERVAL_IN_MS;
}
```

И, наконец, асинхронная функция `performMeasurement()` вызывает API, записывает результат и планирует следующее измерение.

```javascript
async function performMeasurement() {
  // 1. Вызов функции performance.measureUserAgentSpecificMemory().
  let result;
  try {
    result = await performance.measureUserAgentSpecificMemory();
  } catch (error) {
    if (error instanceof DOMException && error.name === 'SecurityError') {
      console.log('Этот контекст небезопасен.');
      return;
    }
    // Повторное создание других ошибок.
    throw error;
  }
  // 2. Запись результата.
  console.log('Memory usage:', result);
  // 3. Планирование следующего измерения.
  scheduleMeasurement();
}
```

Результат может выглядеть следующим образом:

```javascript
// Вывод в консоль:
{
  bytes: 60_000_000,
  breakdown: [
    {
      bytes: 40_000_000,
      attribution: [
        {
          url: "https://foo.com",
          scope: "Window",
        },
      ]
      types: ["JS"]
    },
    {
      bytes: 0,
      attribution: [],
      types: []
    },
    {
      bytes: 20_000_000,
      attribution: [
        {
          url: "https://foo.com/iframe",
          container: {
            id: "iframe-id-attribute",
            src: "redirect.html?target=iframe.html",
          },
        },
      ],
      types: ["JS"]
    },
  ]
}
```

Для возврата значения оценки общего использования памяти используется поле `bytes`. Для значения количества байтов используется [синтаксис чисел с разделителями](https://v8.dev/features/numeric-separators). Это значение сильно зависит от реализации, и его нельзя сравнивать для разных браузеров. Оно может быть разным даже для различных версий одного и того же браузера. На этапе Origin Trials это значение включает показатель использования памяти JavaScript в главном окне, а также во всех элементах iframe **одного и того же сайта** и в связанных окнах. В окончательной версии API в этом значении будет учитываться память JavaScript и DOM для всех элементов iframe, связанных окон и рабочих веб-процессов в текущем процессе. Обратите внимание, что если включена функция [Site Isolation](https://www.chromium.org/Home/chromium-security/site-isolation) (Изоляция сайтов), API не измеряет память [внепроцессных элементов iframe](https://www.chromium.org/developers/design-documents/oop-iframes) между сайтами.

В списке `breakdown` содержится дополнительная информация об используемой памяти. Каждая запись описывает определенную часть памяти и связывает ее с набором окон, элементов iframe и рабочих процессов, идентифицируемых с помощью URL-адресов. В поле `types` указаны типы памяти, зависящие от реализации и связанные с памятью.

Важно обрабатывать все списки общим способом, а не жестко задавать допущения, основанные на конкретном браузере. Например, одни браузеры могут возвращать пустое поле `breakdown` или `attribution`. Другие браузеры могут возвращать несколько записей в поле `attribution`, тем самым сообщая, что им не удается определить, какой из этих записей принадлежит память.

## Обратная связь {: #feedback }

Группа [Web Performance Community Group](https://www.w3.org/community/webperfs/) и команда разработчиков Chrome будут рады, если вы поделитесь своими мыслями и расскажете об опыте использования функции `performance.measureUserAgentSpecificMemory()`.

### Расскажите, что вы думаете о конструкции API

Есть ли в API элементы, не работающие должным образом? Может в нем нет свойств, необходимых для реализации вашей идеи? Сообщите о проблеме, связанной с техническими характеристиками, в [репозитории performance.measureUserAgentSpecificMemory() на GitHub](https://github.com/WICG/performance-measure-memory/issues) или изложите свои мысли о какой-либо существующей проблеме.

### Сообщите о проблеме, связанной с реализацией

Нашли ошибку в реализации для Chrome? Или реализация отличается от спецификации? Сообщите об ошибке на сайте [new.crbug.com](https://bugs.chromium.org/p/chromium/issues/entry?components=Blink%3EPerformanceAPIs). Обязательно укажите как можно больше сведений, предоставьте простые инструкции по воспроизведению ошибки и задайте для параметра **Components** значение `Blink>PerformanceAPIs`. Для быстрого и простого обмена инструкциями по воспроизведению ошибок отлично подходит ресурс [Glitch](https://glitch.com/).

### Продемонстрируйте поддержку

Планируете ли вы использовать функцию `performance.measureUserAgentSpecificMemory()`? Благодаря вашей публичной поддержке команда разработчиков Chrome сможет правильно расставлять приоритеты при работе над функциями и продемонстрирует другим поставщикам браузеров, насколько важна их поддержка. Отправьте твит в аккаунт [@ChromiumDev](https://twitter.com/chromiumdev) и сообщите нам, где и как вы используете эту функцию.

## Полезные ссылки {: #helpful }

- [Эксплейнер](https://github.com/WICG/performance-measure-memory)
- [Демонстрация](https://performance-measure-memory.glitch.me/) | [Исходный код демонстрации](https://glitch.com/edit/#!/performance-measure-memory?path=script.js:1:0)
- [Испытания по схеме Origin Trials](https://developers.chrome.com/origintrials/#/view_trial/1281274093986906113)
- [Отслеживание ошибок](https://bugs.chromium.org/p/chromium/issues/detail?id=1049093)
- [Запись на сайте ChromeStatus.com](https://www.chromestatus.com/feature/5685965186138112)

## Благодарности

Большое спасибо Доменику Деникола (Domenic Denicola), Йоаву Вайссу (Yoav Weiss) и Матиасу Биненсу (Mathias Bynens) за обзоры структуры API, а также Доминику Инфюру (Dominik Inführ), Ханнесу Пайеру (Hannes Payer), Кентаро Хара (Kentaro Hara) и Майклу Липпауцу (Michael Lippautz) за обзоры кода в Chrome. Я также благодарю Пера Паркера (Per Parker), Филиппа Вайса (Philipp Weis), Ольгу Беломестных, Мэтью Болохана (Matthew Bolohan) и Нила Маккея (Neil Mckay) за ценные отзывы пользователей, которые позволили значительно улучшить этот API.

В качестве [главного изображения](https://unsplash.com/photos/5tLfQGURzHM) использовано фото [Харрисона Бродбента](https://unsplash.com/@harrisonbroadbent) (Harrison Broadbent) с [Unsplash](https://unsplash.com)