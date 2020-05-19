---
id: concurrent-mode-intro
title: Знайомство з паралельним режимом (Експериментальний)
permalink: docs/concurrent-mode-intro.html
next: concurrent-mode-suspense.html
---

<style>
.scary > blockquote {
  background-color: rgba(237, 51, 21, 0.2);
  border-left-color: #ed3315;
}
</style>

<div class="scary">

>Увага:
>
>На сторінці описані **експериментальні функції, [яких ще немає](/docs/concurrent-mode-adoption.html) в стабільній версії**. Не використовуйте експериментальні збірки React в продакшн додатках. Ці функції можуть суттєво змінитися та без попередження потрапити в React.
>
>Ця документація орієнтована на першопрохідців та зацікавлених користувачів. **Якщо ви новачок в React, не турбуйтеся про ці функції**, немає необхідності вивчати їх прямо зараз.

</div>

На цій сторінці представлений теоретичний огляд "Паралельного Режиму". **Для більш практичного застосування ви можете ознайомитись з наступними розділами:**

* [Затримка при запиті даних](/docs/concurrent-mode-suspense.html) описує новий механізм запиту даних у React-компонентах.
* [Патерни паралельного UI](/docs/concurrent-mode-patterns.html) показує деякі патерни UI, які стали можливими завдяки паралельному режиму та затримці.
* [Використання паралельного режиму](/docs/concurrent-mode-adoption.html) пояснює, як ви можете спробувати паралельний режим у своєму проекті.
* [Довідник API паралельного режиму](/docs/concurrent-mode-reference.html) документує нові API, доступні в експериментальних версіях.

## Що таке паралельний режим? {#what-is-concurrent-mode}

Паралельний режим — набір нових функцій, які допомагають React додаткам залишатися чутливими та плавно підлаштовується під можливості пристрою користувача та швидкість мережі.

Ці особливості досі експериментальні й можуть змінюватися. Вони ще не є частиною стабільної версії React, але ви можете спробувати їх в експериментальній збірці.

## Блокування проти переривання рендерингу {#blocking-vs-interruptible-rendering} 

**Щоб пояснити паралельний режим, ми будемо використовувати контроль версій як метафору.** Якщо ви працюєте в команді, ви, ймовірно, використовуєте систему контролю версій на зразок Git й працюєте з гілками. Коли гілка готова, ви можете злити свою роботу в master, щоб інші люди могли її витягнути.

До того, як створили контроль версій, робочий процес розробки дуже відрізнявся. Там не було поняття гілок. Якщо ви хотіли відредагувати якісь файли, вам доводилося говорити всім не торкатися цих файлів, поки ви не закінчите роботу. Ви навіть не могли почати працювати над ними одночасно з іншою людиною - ви були буквально *заблоковані*.

Це ілюструє як типово сьогодні працюють UI-бібліотеки, включаючи React. Як тільки вони починають рендерити оновлення, включаючи створення нових вузлів DOM та запуск коду всередині компонентів, вони не можуть перервати цю роботу. Цей підхід ми будемо називати "блокуванням рендеренгу".

У паралельному режимі рендеринг не блокується. Він переривається. Це робить користування додатком зручнішим. Він також розблоковує нові функції, які раніше були неможливі. Перш ніж ми розглянемо конкретні приклади в [наступних](/docs/concurrent-mode-suspense.html) [розділах](/docs/concurrent-mode-patterns.html), ми зробимо загальний огляд нових функцій.

### Переривання рендерингу

Розглянемо список продуктів, що фільтруються. Ви коли-небудь фільтрували список та відчували, що він затиняїться при кожному натисканні клавіш? Деяка робота над оновленням списку продуктів може бути неминучою, наприклад, створення нових вузлів DOM або будування макета браузером. Однак *коли* і *як* ми виконуємо цю роботу, грає велику роль.

Поширений спосіб обійти запинання — не обробляти вхідні дані при кожній зміні (debounce). У такому разі ми оновлюємо список лише *після* того, як користувач перестає друкувати. Однак, може бути неприємно, що інтерфейс користувача не оновлюється під час введення тексту. Як альтернатива, ми могли б "гальмувати" (throttle) введення даних та оновлювати список з певною максимальною частотою. Але потім на пристроях з меншою потужністю ми все-таки почнемо затинатися. Обидва підходи створюють неоптимальний UI.

Причина затинання проста: після початку рендеринга, він не може бути перерваний. Тому браузер не може оновити введення тексту відразу після натискання клавіши. Незалежно від того, наскільки добре може виглядати UI-бібліотека (наприклад, React) у порівнянні з іншими, якщо він використовує блокування рендерингу, певна кількість роботи у ваших компонентах завжди призведе до затинання. І найчастіше це не так просто виправити.

**Паралельний режим усуває це фундаментальне обмеження, роблячи рендеринг переривчастим.** Це означає, що коли користувач натискає іншу клавішу, React-у не потрібно блокувати браузер для оновлення введення тексту. Натомість він може дозволити браузеру відобразити оновлення вводу, а потім продовжити оновлювати список *у пам'яті*. Коли рендеринг закінчений, React оновлює DOM, а зміни відображаються на екрані.

Концептуально ви можете думати про це так, що React готує кожне оновлення "на гілці". Так само, як ви можете відмовитися від роботи у гілках або перемикатися між ними, React у паралельному режимі може перервати постійне оновлення, щоб зробити щось важливіше, а потім повернутися до того, що він робив раніше. Ця методика може також нагадувати вам про [подвійну буферизацію](https://wiki.osdev.org/Double_Buffering) у відеоіграх.

Паралельний режим зменьшує потребу в очікуванні (debouncing) та гальмуванні (throttling) в інтерфейсі користувача. Оскільки рендеринг переривається, React не потрібно штучно *затримувати* рендеринг, щоб уникнути затинання. Він може почати рендеринг відразу, але перервати цю роботу, коли це необхідно, щоб додаток завжди був в змозі реагувати на запити.

### Навмисні послідовності завантаження {#intentional-loading-sequences}

Ми вже говорили, що паралельний режим у React — це ніби працювання "на гілці". Гілки корисні не тільки для короткочасних виправлень, але і для довготривалих змін. Іноді ви можете працювати над функцією, але може пройти кілька тижнів, перш ніж вона виявиться в «досить хорошому стані», щоб злитися з master. Цей аспект метафори про управління версіями стосується і рендерингу. 

Уявіть, що ми пересуваємося між двома екранами в додатку. Іноді у нас може не вистачати завантаженого коду і даних, щоб показати користувачеві «досить добрий» стан завантаження на новому екрані. Перехід на порожній екран або на великий спінер може бути неприємним досвідом. Проте найчастіше, щоб отримати необхідний код та дані непотрібно занадто багато часу. **Чи не було б приємніше, якби React міг залишитися на старому екрані трохи довше і "пропустити" "поганий стан завантаження" перед тим, як показати новий екран?**

Незважаючи на те, що це можно робити і сьогодні, це важко організувати. У паралельному режимі ця функція вбудована. React спочатку починає готувати новий екран в пам'яті - або, як йдеться у нашій метафорі, "на іншій гілці". Тож React може зачекати, перш ніж оновити DOM, щоб завантажувати більше контенту. У паралельному режимі ми можемо сказати React продовжувати показувати старий екран, повністю інтерактивний, із вбудованим індикатором завантаження. І коли новий екран буде готовий, React може перевести нас до нього.

### Паралельність {#concurrency}

Давайте резюмуємо два приклади, наведені вище, і подивимося, як паралельний режим об'єднує їх: **У паралельному режимі React може працювати над кількома оновленнями стану *паралельно*** - так само, як гілки дозволяють різним членам команди працювати самостійно:

* Для  оновлень пов'язаних до ЦП (наприклад, створення вузлів DOM та запуску коду компонента) паралельність означає, що більш термінове оновлення може «перервати» рендеринг, що вже розпочався.
* Для оновлень пов'язаних до вводу-виводу (таких як отримання кода або даних з мережі), паралельність означає, що React може почати візуалізацію в пам'яті ще до того, як всі дані надійдуть, і пропустити показ порожніх станів завантаження.

Важливо, те, що ви *використовуєте* React так само. Поняття, такі як компоненти, реквізити та стан, принципово працюють однаково. Коли ви хочете оновити екран, ви встановлюєте стан.

React використовує евристику, щоб вирішити, наскільки "терміновим" є оновлення, і дозволяє вам налаштувати його за допомогою декількох рядків коду, щоб ви могли досягти бажаного досвіду користувача для кожної взаємодії.

## Досвід впровадження у продакшн {#putting-research-into-production}

Існує загальна тема щодо можливості паралельного режиму. **Його місія полягає в тому, щоб допомогти інтегрувати результати від дослідження взаємодії людини і комп'ютера в реальному UI.**

Наприклад, дослідження показують, що відображення занадто багатьох станів проміжного завантаження при переході між екранами робить відчуття перехіду *повільніше*. Ось чому паралельний режим показує нові стани завантаження за фіксованим "графіком", щоб уникнути нестабільності та занадто частого оновлення.

Аналогічно, з досліджень ми знаємо, що з такі взаємодії, як наведення курсора та введення тексту, потрібно обробляти за дуже короткий проміжок часу, тоді як кліки та переходи сторінок можуть зачекати трохи довше, не відчуваючи лага. Різні "пріоритети", які використовує паралельний режим, приблизно відповідають категоріям взаємодії в дослідженні людського сприйняття.

Команди з сильним фокусом на досвіді користувачів іноді вирішують подібні проблеми одноразовими рішеннями. Однак ці рішення рідко виживають довгий час, оскільки їх важко підтримувати. У паралельному режимі наша мета полягає в тому, щоб визначити результати досліджень інтерфейсу в самій абстракції та надати ідіоматичні способи їх використання. Як UI-бібліотека, React чудово підходить для цього.

## Наступні кроки {#next-steps}

Тепер ви знаєте, що таке паралельний режим!

На наступних сторінках ви дізнаєтесь більше деталей щодо конкретних тем:

* [Затримка при запиті даних](/docs/concurrent-mode-suspense.html) описує новий механізм запиту даних у React-компонентах.
* [Патерни паралельного UI](/docs/concurrent-mode-patterns.html) показує деякі патерни UI, які стали можливими завдяки паралельному режиму та затримці.
* [Використання паралельного режиму](/docs/concurrent-mode-adoption.html) пояснює, як ви можете спробувати паралельний режим у своєму проекті.
* [Довідник API паралельного режиму](/docs/concurrent-mode-reference.html) документує нові API, доступні в експериментальних версіях.