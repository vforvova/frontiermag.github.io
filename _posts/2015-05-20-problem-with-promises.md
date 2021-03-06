---
layout: post
tag: 'javascript'
title: Проблема с промисами
short: "Пора признать, что у нас с ними проблема: мы используем
их, не понимая"
published: true
featured: true
author: Нолан Лоусон
---
Дорогие джаваскриптеры, пора признать: у нас проблема с промисами.

Нет, не промисами как таковыми. [Спецификация A+](https://promisesaplus.com/) превосходна.

А с тем, что я вижу множество программистов, не справляющихся с API PouchDB
и другими API, построенных на промисах. Их беда в том, что
<em style="font-size: 2em; display: block; text-align: center; margin:
1em;">Многие из нас используют промисы, не до конца их понимая.</em>

В это сложно поверить, но посмотрите на задачку, которую я недавно
[запостил в твиттер](https://twitter.com/nolanlawson/status/578948854411878400):

<blockquote><p> В чём разница между этими промисами?</p>
 {% highlight javascript %}
 doSomething().then(function () {
	return doSomethingElse();
 });

 doSomething().then(function () {
	doSomethingElse();
 });

doSomething().then(doSomethingElse());

doSomething().then(doSomethingElse);
 {% endhighlight %}
</blockquote>

Если вы знаете ответ, то поздравляю: вы мастер промисов. Можете
дальше не читать эту статью.


Для остальных 99.99% из вас, вы не одиноки. Никто из ответивших на мой
твит не смог решить эту задачу, а ответ на #3 меня и самого
удивил. Да, несмотря на то, что я сам написал этот тест!

Ответы приведены в конце, но сначала, я бы хотел исследовать, почему
промисы вообще такие каверзные, и почему столь многие из нас &mdash;
и&nbsp;новички, и&nbsp;эксперты &mdash; попадаются на эту удочку. Я также хочу
предложить то, что мне кажется моментом прозрения, _один хитрый трюк_,
чтобы влёгкую понять промисы. И да, я правда считаю, что они не
такие уж и сложные!

## Итак, промисы?

В текстах про них вы часто найдёте отсылки к
[пирамиде зла](https://medium.com/@wavded/managing-node-js-callback-hell-1fe03ba8baf),
со страшным коллбэчным кодом, который уходит за правую часть экрана.

Промисы действительно решают эту проблему, но они не только про
отступы. Как нам рассказали в великолепном выступлении
&laquo;[Избавление от ада коллбэков](http://youtu.be/hf1T_AONQJU)&raquo;,
настоящая беда с коллбэками в том, что они отбирают у нас такие вещи,
как `return` и `throw`. Вместо этого, весь поток нашей программы
основан на _сайд-эффектах_: одна функция невзначай вызывает другую.

На самом деле, коллбэки ещё коварнее: они отбирают у нас _стэк_,
который мы принимаем за должное в программировании.
Писать код без доступа к стэку всё равно что ехать на машине
без педали тормоза: не понять, как сильно она нужна,
пока вы не потянетесь за ней, а её там нет.

Вся суть промисов в том, чтобы вернуть основы языка, которые
мы утратили, когда стали писать асинхронно: `return`, `throw`
и стэк. Но надо знать, как их правильно использовать, чтобы
получить от них толк.

## Ошибки новичков

Некоторые пытаются объяснить промисы [картинками](http://andyshora.com/promises-angularjs-explained-as-cartoon.html)
или представляя их предметом: <q>О, это штука, которую можно передавать
тут и там, она представляет асинхронное значение.</q>

Мне кажется, это не помогает. Промисы &mdash; про структуру кода и поток.
Поэтому я думаю, что лучше просто пройтись
по распространенным ошибкам и показать, как их исправить.
Я называю их ошибками новичка в том смысле, что: <q>ты новичок сейчас, но скоро
станешь профи</q>.

<aside style="background-color: aliceblue; padding: 1em; margin: 2em -1em;">
<span style="font-variant: small-caps;">Лирическое отступление</span>: &laquo;промисы&raquo;
означают для разных людей много разных вещей, но в этой статье я буду говорить о них,
подразумевая <a href="https://promiseaplus.com">официальную спецификацию</a>,
реализованную в современных браузерах как <code>window.Promise</code>.
Не во всех браузерах есть <code>window.Promise</code>, так что для хорошего полифилла
посмотрите на библиотеку с остроумным названием <a href="https://github.com/calvinmetcalf/lie">Lie</a>,
чуть ли не самую маленькую из соответствующих спецификации.
</aside>

### Ошибка новичка #1: пирамида зла из промисов

API PouchDB по большей части построено на промисах, и я вижу 
много плохих паттернов, связанных с ними.

Самое распространенное плохое решение вот это:

{% highlight javascript %}
remotedb.allDocs({
    include_docs: true,
    attachments: true
}).then(function (result) {
    var docs = result.rows;
    docs.forEach(function(element) {
        localdb.put(element.doc).then(function(response) {
          alert("Pulled doc  with id " + element.doc._id + " and added to local db.");
        }).catch(function (err) {
            if (err.status == 409) {
                localdb.get(element.doc._id).then(function (resp) {
                    localdb.remove(resp._id, resp._rev).then(function (resp) {
  // И так далее.. Да, можно использовать промисы 
  // словно коллбэки, и да, это всё равно, что подстригать
  // ногти циркулярной пилой
{% endhighlight %}

Если вы думаете, что такие ошибки случаются только с новичками, 
вас удивит, что я взял этот [фрагмент кода](https://github.com/blackberry/BB10-WebWorks-Community-Samples/blob/d6ee75fe23a10d2d3a036013b6b1a0c07a542099/pdbtest/www/js/index.js)
из [официального девелоперского блога BlackBerry](http://devblog.blackberry.com/2015/05/connecting-to-couchbase-with-pouchdb/)!
Старые привычки коллбэков отмирают не сразу. 

<aside>(Разработчику: извини, что придрался к твоему коду,
но этот пример очень показателен.)</aside>

Этот стиль гораздо лучше:
{% highlight javascript %}
remotedb.allDocs(...).then(function (resultOfAllDocs) {
    return localdb.put(...); 
}).then(function (resultOfPut) {
    return localdb.get(...);
}).then(function (resultOfGet) {
     return localdb.put(...);
}).catch(function (err) {
    console.log(err);
});
{% endhighlight %}

Это называется _композиция промисов_, и это одна из их великих супер-возможностей.
Каждая функция будет вызвана только тогда, когда зарезолвится
предыдущий промис, и она будет вызвана с результатом этого промиса.
Подробнее ниже.


### Ошибка новичка #2: WTF, как мне сделать forEach() по промисам?

Здесь у большинства людей ломается понимание промисов. Как только они
тянутся к знакомому циклу `forEach()` (или циклу `for`, или `while`),
у них нет идей, как заставить их работать с промисами. Так что они
пишут что-то вроде:

{% highlight javascript %}
// Надо удалить все документы через remove()
db.allDocs({
    include_docs: true
}).then(function (result) {
    result.rows.forEach(function (row) {
        db.remove(row.doc);  
    });
}).then(function () {
    // Я наивно верю, что все документы
    // были уже удалены через `remove()`!
}); 

{% endhighlight %}

Что не так с этим кодом? То, что первая функция вернёт
`undefined`, а это значит, что вторая функция не будет дожидаться,
пока `db.remove()` будет вызвана на всех документах. На самом деле,
она вообще ничего не будет ждать и может быть выполнена с любым
количеством удаленных документов!

И это особенно коварный баг, потому что вы можете не заметить,
что что-то не так, если PouchDB удалит документы быстрее,
чем обновляется ваш интерфейс. Баг может всплыть только в странных
race conditions или в определенных браузерах, и тогда его будет
практически невозможно отдебажить.

<abbr title="Too long; Didn't Read">TLDR</abbr> всего этого в том,
что `forEach()`/`for`/`while` это не те конструкции, что вы ищете.
Вам нужен `Promise.all()`:

{% highlight javascript %}
db.allDocs({include_docs: true}).then(function (result) {
    return Promise.all(result.rows.map(function (row) {
        return db.remove(row.doc);
    });
}).then(function (arrayOfResults) {
    // Все документы удалены на момент вызова этой функции!
});
{% endhighlight %}

Что только что произошло? `Promise.all()` берёт _массив промисов_ на вход,
а затем возвращает другой промис, который исполнится тоько тогда,
когда зарезолвится каждый из этих промисов. Это асинхронный
эквивалент цикла `for`.

`Promise.all()` также передаёт массив результатов следующей функции,
что зачастую пригождается, например, если вы хотите сделать `get()`
сразу нескольких вещей из PouchDB. Промис `all()` также будет отклонен,
если _любой из под-промисов будет отклонен_, что ещё полезнее.

### Ошибка новичка #3: забыть добавить `.catch()`

Это другая распространенная ошибка. Блаженно верующие в то, что
их промисы никогда не отклоняются, многие разработчики забывают
добавить `.catch()` где-либо в своем коде. К несчастью, это приведет
к тому, что любая брошенная ошибка будет _проглочена_, 
и вы даже не увидите её в консоли. Это реально сложно дебажить.

Чтобы избежать этого невеселого сценария, я приобрел привычку просто
добавлять следующий код к цепочке из промисов:

{% highlight javascript %}
somePromise().then(function () {
    return anotherPromise();
}).then(function () {
    return yetAnotherPromise();
}).catch(console.log.bind(console)); // <-- ага, вот эти ребята
{% endhighlight %}

Даже если вы не ждёте ошибки, мудрым решением будет всегда добавить `catch()`.
Ваша жизнь станет проще, особенно если ваши предположения окажутся
неверными.

### Ошибка новичка #4: использовать deferred

Эту ошибку я вижу [всё время](http://gonehybrid.com/how-to-use-pouchdb-sqlite-for-local-storage-in-your-ionic-app/),
и я не хочу даже повторять это здесь,
из-за страха, что подобно Битлджусу, просто назвать имя &mdash; значит
вызвать ещё больше его же.

Вкратце, у промисов долгая и бурная история, и на то, чтобы сделать всё правильно,
у сообщества джаваскриптеров ушло много времени. В ранние дни,
jQuery и Angular использовали повсюду паттерн deferred, который сейчас
был заменен спецификацией ES6 Promise, которую реализуют &laquo;хорошие&raquo;
библиотеки, такие как Q, When, RSVP, Bluebird, Lie и другие.

Так что если это слово встречается в вашем коде (я не стану повторять его
в третий раз!), то вы делаете что-то не так. Вот как избежать этого.

Во-первых, многие библиотеки дают вам способ &laquo;импортировать&raquo;
промисы от других библиотек. Например, модуль `$q` из Ангуляра позволяет
вам обернуть не-`$q`шные промисы через `$q.when()`. Так что пользователи
Ангуляра могут обернуть промис от PouchDB таким образом:

{% highlight javascript %}
$q.when(db.put(doc)).then(/* ... */); // <-- это всё!
{% endhighlight %}

Другой способ &mdash; использовать [паттерн раскрывающегося конструктора](https://blog.domenic.me/the-revealing-constructor-pattern/),
что удобно для оборачивания API без промисов. Например, чтобы обернуть
основанное на коллбэках API вроде `fs.readFile()` из node.js, нужно просто:

{% highlight javascript %}
new Promise(function (resolve, reject) {
    fs.readFile('myfile.txt', function (err, file) {
        if (err) { return reject(err); }
        resolve(file); });
}).then(/* ... */)
{% endhighlight %}

Готово! Мы победили страшный def... Ха, поймал себя. :)


### Ошибка новичка #5: использовать сайд-эффекты вместо возврата значения

Что не так с этим кодом?

{% highlight javascript %}
somePromise().then(function () {
    someOtherPromise();
}).then(function () { 
  // Фух, надеюсь someOtherPromise() разрешилось!
  // Спойлер: нет, не разрешилось.
});
{% endhighlight %}

Хорошо, пришло время рассказать всё, что вам нужно знать об промисах.

Серьёзно, это _один странный трюк_, который, когда вы его поймёте,
предотвратит все ошибки, о которых я говорил выше. Готовы?

Я говорил, что магия промисов в том, что они возвращают нам обратно
драгоценные `return` и `throw`. Но как это выглядит на практике?

Каждый промис даёт вам метод `then()` (или `catch(), который просто
сахар для <code>then(null, ...)</code>). Вот мы внутри функции <code>then()</code>:

{% highlight javascript %}
somePromise().then(function () {
   // Я внутри then()! 
});
{% endhighlight %}

Что можно здесь сделать? Три вещи:

1. вернуть другой промис через `return`
2. вернуть синхронное значение через `return` (или `undefined`)
3. бросить синхронную ошибку через `throw`

<p style="margin-top: 1.5em;">Это всё. Как только вы поймёте этот трюк, вы поймёте промисы.
Поэтому давайте пройдёмся по каждому пункту по отдельности.</p>

#### 1. Вернуть другой промис

Это распространненый паттерн, который можно найти во многих текстах про промисы,
как в примере про &laquo;композицию промисов&raquo; выше.

{% highlight javascript %}
getUserByName('nolan').then(function (user) {
    return getUserAccountById(user.id);
}).then(function (userAccount) {
    // У меня есть аккаунт!
});
{% endhighlight %}

Отметьте, что я возвращаю второй промис &mdash; этот `return` очень важен.
Если бы его не было, то `getUserAccountById()` был бы _сайд-эффектом_
и следующая функция получила бы `undefined` вместо `userAccount`.


#### 2. Вернуть синхронное значение (или `undefined`)

Возвращение `undefined` зачастую является ошибкой, но возвращение синхронного
значения &mdash; отличный способ превратить синхронный код
в код на промисах. Скажем, у нас есть кэш пользователей в памяти. Мы можем:

{% highlight javascript %}
getUserByName('nolan').then(function (user) {
    if (inMemoryCache[user.id]) {
        return inMemoryCache[user.id]; // возвращаем синхронное значение
    }
    return getUserAccountById(user.id); // возвращаем промис
}).then(function (userAccount) {
    // У меня есть аккаунт!
});
{% endhighlight %}

Клёво, правда? Второй функции без разницы, был ли `userAccount`
получен синхронно или асинхронно, а первая может вернуть либо
синхронное, либо асинхронное значение.

Неудобная неприятность в том, что функции без `return`
в Javascript, строго говоря, возвращают `undefined`, а это значит, что
легко случайно привнести сайд-эффекты тогда, когда вы хотели
вернуть что-то другое.

Поэтому, я привык всегда _возвращать что-нибудь или кидать ошибку из
`then()`_, и вам рекомендую то же самое.


#### 3. Бросить синхронную ошибку

Кстати, о `throw`, здесь промисы становятся ещё лучше. Скажем, мы
хотим бросить синхронную ошибку, если пользователь вышел из аккаунта.
Это легко:

{% highlight javascript %}
getUserByName('nolan').then(function (user) {
    if (user.isLoggedOut()) {
        throw new Error('user logged out!'); // кидаем ошибку!
    }
    if (inMemoryCache[user.id]) {
        return inMemoryCache[user.id]; // возвращаем простое значение!
    }
    return getUserAccountById(user.id); // возвращаем промис!
}).then(function (userAccount) {
   // У меня есть аккаунт!
}).catch(function (err) {
   // Блин, а у меня ошибка! :(
});
{% endhighlight %}

Наш `catch()` получит синхронную ошибку, если пользователь вышел из аккаунта,
а также если _любой из промисов будет отклонен_. Опять-таки, функции без разницы,
была ли ошибка синхронной или асинхронной.

Это особенно полезно, потому что может помогает находить ошибки
в процессе разработчки. Например, если где-то внутри `then()` мы делаем
`JSON.parse()`, который кидает синхронную ошибку, если JSON не корректен.
С коллбэками, ошибка была бы проглочена, но с промисами мы можем просто
обработать её внутри `catch()`.



## Продвинутые ошибки

Отлично, теперь вы узнали трюк для понимания промисов, давайте поговорим
о редких проблемах. Потому что, конечно, всегда есть редкие проблемы.

Эти ошибки я определяю как &laquo;продвинутые&raquo;, потому что видел их
только у программистов, которые уже довольно неплохо прониклись промисами.
Но нам нужно их обсудить, чтобы решить задачку из начала этой статьи.

### Продвинутая ошибка #1: не знать о `Promise.resolve()`

Как я показал выше, промисы удобны для превращения синхронного кода
в асинхронный. Как бы то ни было, если вы часто пишете код вроде:

{% highlight javascript %}
new Promise(function (resolve, reject) {
      resolve(someSynchronousValue);
}).then(/* ... */);
{% endhighlight %}

То вы можете выразить это лаконичнее, используя `Promise.resolve()`:

{% highlight javascript %}
Promise.resolve(someSynchronousValue).then(/* ... */);
{% endhighlight %}

Также этот метод невероятно полезен для ловли синхронных ошибок.
Настолько полезен, что я привык писать в начале почти всех
моих методов API, возвращающих промисы, что-то вроде:

{% highlight javascript %}
function somePromiseAPI() { 
    return Promise.resolve().then(function () {
        doSomethingThatMayThrow();
        return 'foo';
    }).then(/* ... */);
}

{% endhighlight %}

Просто запомните: код, который может кидать синхронные ошибки, &mdash;
хороший кандидат для ошибок, которые почти невозможно отдебажить из-за того,
что они проглочены где-нибудь по дороге. Но если оборачивать всё в `Promise.resolve()`,
то всегда можно будет поймать их в `catch()` позже.

Аналогично, есть `Promise.reject()`, который возвращает промис, который
немедленно отклоняется:

{% highlight javascript %}
Promise.reject(new Error('ой всё'));
{% endhighlight %}

### Продвинутая ошибка #2: `catch()` немного не `then(null, ...)`

Выше я говорил, что `catch()` &mdash; просто синтаксический сахар.
Так что эти два фрагмента эквивалентны:

{% highlight javascript %}
somePromise().catch(function (err) {
    // обрабатываем ошибку...
});

somePromise().then(null, function (err) {
    // обрабатываем ошибку
});
{% endhighlight %}

Но это не значит, что эти два фрагмента одинаковы:

{% highlight javascript %}
somePromise().then(function () {
    return someOtherPromise();
}).catch(function (err) {
    // обрабатываем ошибку
});

somePromise().then(function () {
    return someOtherPromise();
}, function (err) {
    // обрабатываем ошибку
}); 
{% endhighlight %}

Если вам интересно, почему они не одинаковые, подумайте, что произойдет,
если первая функция бросит ошибку:

{% highlight javascript %}
somePromise().then(function () {
    throw new Error('oh noes');
}).catch(function (err) {
    // Поймал! :)
});

somePromise().then(function () {
    throw new Error('oh noes');
},function (err) {
    // Не словил! :(
});
{% endhighlight %}

Когда вы используете формат `then(resolveHandler, rejectHandler)`,
`rejectHandler` не поймает ошибку, если её кинет `resolveHandler`.

Поэтому я просто не использую второй аргумент у `then()` и всегда предпочитаю
`catch()`. Я делаю исключение только для случаев, когда я пишу асинхронные тесты
для [Mocha](http://mochajs.org/), где я могу написать тест на то,
что ошибка действительно кидается:

{% highlight javascript %}
it('should throw an error', function () {
    return doSomethingThatThrows().then(function () {
        throw new Error('I expected an error!');
    }, function (err) {
        err.should.exist();
    });
});
{% endhighlight %}

И раз мы уж тут, Mocha и [Chai](http://chaijs.com/) образуют хорошую комбинацию для
тестирования API с промисами. В проекте [pouchdb-plugin-seed](https://github.com/pouchdb/plugin-seed) 
есть несколько [шаблонных тестов](https://github.com/pouchdb/plugin-seed/blob/master/test/test.js),
чтобы начать было легче.

### Продвинутая ошибка #3: промисы или фабрики промисов

Скажем, вы хотите исполнить несколько промисов один за другим последовательно.
То есть, вам нужно что-то вроде `Promise.all()`, но которое
не выполняет промисы параллельно.

Наивный код будет выглядеть как-то так:
{% highlight javascript %}
function executeSequentially(promises) {
    var result = Promise.resolve();
    promises.forEach(function (promise) {
        result = result.then(promise);
    });
    return result;
}
{% endhighlight %}

К несчастью, он не будет работать так, как вы предполагали.
Промисы, которые вы передадите в `executeSequentially()` всё ещё
будут исполняться параллельно.

Всё потому, что вам не нужно оперировать массивом промисов
вовсе. По спецификации, как только промис создан, он начинает
исполняться. Так что на самом деле вы хотите массив _фабрик промисов_:

{% highlight javascript %}
function executeSequentially(promiseFactories) {
    var result = Promise.resolve();
    promiseFactories.forEach(function(promiseFactory) {
        result = result.then(promiseFactory);
    });
    return result;
} 
{% endhighlight %}

Я знаю, что вы сейчас подумали: &laquo;Кто пустил сюда джава-программиста,
и почему он говорит о фабриках?&raquo;. Но фабрика промисов это просто
функция, которая возвращает промис:

{% highlight javascript %}
function myPromiseFactory() {
    return somethingThatCreatesAPromise();
}
{% endhighlight %}

Почему это работает? Потому что фабрика не создаёт промис до тех пор,
пока её не попросят. Работает так же, как и функция `then` &mdash; в сущности,
это одно и то же!

Если вы посмотрите на функцию `executeSequentially()`, а затем представите,
что `myPromiseFactory` подставляется внутрь `result.then(...)`, то, надеюсь,
у вас зажгётся лампочка над головой. В этот момент вы достигли просветления промисов.


### Продвинутая ошибка #4: ладно, а что если я хочу результат двух промисов?

Часто один промис зависит от другого, но нам нужен результат обоих.
Например:

{% highlight javascript %}
getUserByName('nolan').then(function (user) {
    return getUserAccountById(user.id);
}).then(function (userAccount) {
    // чёрт, объект user мне тоже нужен!
});
{% endhighlight %}

Избегая пирамиды зла из желания быть хорошими разработчиками, мы можем
просто хранить объект `user` в переменной во внешней области видимости:

{% highlight javascript %}
var user;
getUserByName('nolan').then(function (result) {
    user = result;
    return getUserAccountById(user.id);
}).then(function (userAccount) { 
    // окей, здесь доступны и `user` и `userAccount`
});
{% endhighlight %}

Это работает, но мне кажется немного неуклюжим. Я рекомендую следующее:
отбросьте ваши предубеждения и поддержите пирамиду:

{% highlight javascript %}
getUserByName('nolan').then(function (user) {
    return getUserAccountById(user.id).then(function (userAccount) {
        // окей, есть доступ и к `user` и к `userAccount`
    });
});
{% endhighlight %}

...хотя бы, временно. Если отступы снова когда-нибудь станут проблемой,
можно будет сделать то, что джаваскриптеры делают с незапамятных времен,
и вынести анонимную функцию в именованную:

{% highlight javascript %}
function onGetUserAndUserAccount(user, userAccount) {
    return doSomething(user, userAccount);
}

function onGetUser(user) {
    return getUserAccountById(user.id).then(function (userAccount) {
        return onGetUserAndUserAccount(user, userAccount);
    });
}

getUserByName('nolan')
  .then(onGetUser)
  .then(function () {
     // на тот момент `doSomething()` уже выполнена и мы снова
     // на нулевом отступе
  });
{% endhighlight %}

По мере того, как ваш код с промисами становится всё сложнее,
вы поймёте, что вытаскиваете всё больше и больше функций
в именованные. Это ведёт, на мой взгляд, к эстетически приятному коду,
который выглядит примерно так:

{% highlight javascript %}
putYourRightFootIn()
  .then(putYourRightFootOut)
  .then(putYourRightFootIn)  
  .then(shakeItAllAbout);
{% endhighlight %}

Промисы как раз про это.


### Продвинутая ошибка #5: проваливание промисов

Наконец-то, это та ошибка, которую я допустил в изначальной задаче.
Это очень редкий юзкейс, и он может никогда не появиться в вашем коде,
но он точно удивил меня.

Как вы думаете, что выведет этот код?

{% highlight javascript %}
Promise.resolve('foo').then(Promise.resolve('bar')).then(function (result) {
    console.log(result);
});

{% endhighlight %}

Если вы думаете, что он выведет bar, то вы ошиблись. Он выведет foo!

Так случается потому, что когда вы передаёте в `then()` не функцию
(а, например, промис), то это интерпретируется как `then(null)`,
что заставляет проваливаться дальше результат предыдущего промиса.
Можете сами проверить:

{% highlight javascript %}
Promise.resolve('foo').then(null).then(function (result) {
    console.log(result);
});
{% endhighlight %}

Добавьте сколько вам угодно `then(null)`, оно всё равно выведет foo.

Это возвращает нас к предыдущему пункут про промисы и фабрики промисов.
Вкратце, можно передать промис напрямую в метод `then()`, но результат
будет не тем, каким вы его ожидаете. `then()` предназначен для функций,
так что скорее всего вы имели в виду:

{% highlight javascript %}

Promise.resolve('foo').then(function () {
    return Promise.resolve('bar');
}).then(function (result) {
    console.log(result);
});
{% endhighlight %}

Это выведет bar, как мы и ожидали.

Так что запомните: всегда передавайте функцию в `then()`!

## Решение задачи

Теперь, когда мы узнали всё про промисы (или почти всё!), мы сможем
решить задачу, которую я поставил в начале этого поста.

Вот ответы для каждого из примеров в графическом формате, чтобы было проще
представить:


### Пример #1

{% highlight javascript %}
doSomething().then(function () {
   return doSomethingElse();
}).then(finalHandler);
{% endhighlight %}

Ответ:

<pre class="mono">
doSomething
|-----------------|
                  doSomethingElse(undefined)
                  |------------------|
                                     finalHandler(resultOfDoSomethingElse)
                                     |------------------|
</pre>


### Пример #2
{% highlight javascript %}
doSomething().then(function () {
    doSomethingElse();
}).then(finalHandler);
{% endhighlight %}

Ответ:

<pre class="mono">
doSomething
|-----------------|
                  doSomethingElse(undefined)
                  |------------------|
                  finalHandler(undefined)
                  |------------------|
</pre>

### Пример #3
{% highlight javascript %}
doSomething().then(doSomethingElse())
    .then(finalHandler);
{% endhighlight %}

Ответ:

<pre class="mono">
doSomething
|-----------------|
doSomethingElse(undefined)
|---------------------------------|
                  finalHandler(resultOfDoSomething)
                  |------------------|
</pre>

### Пример #4
{% highlight javascript %}
doSomething().then(doSomethingElse)
    .then(finalHandler);
{% endhighlight %}

Ответ:

<pre class="mono">
doSomething
|-----------------|
                  doSomethingElse(resultOfDoSomething)
                  |------------------|
                                     finalHandler(resultOfDoSomethingElse)
                                     |------------------|
</pre>


Если эти ответы всё ещё кажутся вам странными, то я рекомендую вам перечитать статью ещё раз,
или определить методы `doSomething()` и `doSomethingElse()`  и попробовать их в браузере.

Для ещё более продвинутых использований промисов, проверьте мою [шпаргалку крутых советов](https://gist.github.com/nolanlawson/6ce81186421d2fa109a4).



## Последнее слово про промисы

Они великолепны. Если вы всё ещё используете коллбэки, я
настоятельно рекомендую вам переключиться на промисы. Ваш код станет
меньше, более элегантным и более простым для понимания и обсуждения.

Если вы всё ещё мне не верите, вот доказательство: [рефакторинг модуля
map/reduce в PouchDB](https://t.co/hRyc6ENYGC), чтобы перейти с коллбэков на
промисы. Результат: 290 вставок, 555 удалений.

Внезапно, человек, который написал этот грязный код на коллбэках оказался... мной!
Это послужило мне первым уроком мощи промисов, и я благодарю других контрибьюторов PouchDB
за то, что направляли меня по дороге.

Конечно, промисы не идеальны. Это правда, что они лучше чем коллбэки, но это всё равно
что сказать: удар в живот лучше пинка в зубы. Да, одно предпочтительнее другого,
но будь у вас выбор, лучше избежать и того, и другого.

И хотя они лучше коллбэков, промисы всё равно трудны для понимания и подвержены ошибкам,
о чём свидетельствует то, что я счёл необходимым написать этот пост. Новички и эксперты
одинаково постоянно допускают ошибки, и, честно говоря, это не их вина. Проблема в том,
что промисы хоть и ближе к паттернам, которые мы используем в синхронном коде,
они являются хорошей заменой, а не тем же самым.

В идеале, вы не должны быть обязанными выучить кучу неясных правил и новых API чтобы
делать то, что в синхронном мире замечательно делается при помощи `return`, `catch`,
`throw` и циклов `for`. Не должно быть двух параллельных систем, которые надо всё время
держать в голове.


## Ждём `async`/`await`

Это то, что я показал в статье &laquo;[Приручаем асинхронного зверя в ES7](http://pouchdb.com/2015/03/05/taming-the-async-beast-with-es7.html),
где я исследовал ключевые слова `async`/`await` из ES7 и как они глубже интегрируют промисы в язык.
Вместо того, чтобы писать псевдо-синхронный код (с фейковым `catch()`, который похож
на `catch`, но не слишком), в ES7 мы сможем использовать настоящие
`try`/`catch`/`return`, точь-в-точь как мы научились на первом курсе.

Это большой плюс Javascript как языку. В конце концов, анти-паттерны с промисами
всё равно будут всплывать, если наши инструменты не будут сообщать нам, что мы
допускаем ошибку.

Возьмём пример из истории Javascript, я думаю, что честным будет сказать, что
[JSLint](http://jslint.com/) и [JSHint](http://jshint.com/) оказали большую услугу сообществу,
чем книга [Javascript: The Good Parts](http://amzn.com/0596517742),
хотя в них содержится, в общем-то, одна и та же информация. Разница в том, что в одном случае
тебе говорят какую именно ты допустил ошибку и где, а в другом это просто книга,
где ты пытаешься понять ошибки других.

Прелесть `async`/`await` из ES7 в том, что, по большей части, ошибки будут в виде
неверного синтаксиса или ошибки компиляции, нежели тонких багов во время исполнения.
Но пока эти времена не пришли, всё-таки хорошо понять на что способны промисы,
и как их правильно использовать в ES5 и ES6.

Я осознаю, что этот пост, как и _Javascript: The Good Parts_, будет иметь ограниченное
влияние, но надеюсь, что на него можно будет кинуть ссылку людям, когда вы заметите
у них эти самые ошибки. Потому что многим из нас стоит признать: &laquo;У меня проблема
с промисами&raquo;

<footer style="background: aliceblue; padding: 1em; margin: 0 -1em -1em;">
    <a href="http://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html">Оригинальный пост</a> написан Ноланом Лоусоном, перевод <a href="http://marinintim.com/">Маринина Тимофея</a>.
</footer>
