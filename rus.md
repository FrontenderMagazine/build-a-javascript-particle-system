Создание системы частиц в 200 строчек JavaScript
------------------------------------------------------------

!["система частиц"]["система частиц"]

Меньше чем в 200 строчек чистого JavaScript мы создадим гибкую систему частиц с
несколькими излучателями и полями гравитации, которые будет притягивать или
отталкивать тысячи частиц.

Всё это началось [личным проектом][1] и превратилось в [Chrome-эксперимент][2]
два или три года назад, когда я всерьёз занялся JavaScript-разработкой. Я не
математик, физик и даже не разработчик игр, поэтому наверняка есть более
правильные или эффективные реализации того, о чём я буду писать в этой статье.
Не смотря на всё это, это был отличный способ изучить производительность
JavaScript.

Самое важное, что я вынес из этого проекта, это осознание низкого порога
вхождения в графическую разработку на данный момент. Если у тебя есть текстовый
редактор и браузер, то ты можешь писать привлекательные визуализации и даже
видео-игры *уже сейчас*.

## Настройка окружения

Нет ни одной причины по которой стоило бы глубоко вдаваться в объяснение
canvas-элемента и его 2d-контекста, так как в мире есть [детальные статьи][4],
посвящённые именно этому. Если вам всё же нужна подробная информация, то
выберите [одну из нескольких тысяч][5].

    <!DOCTYPE html>
    <html>
      <body>
        <canvas></canvas>
      </body>
    </html>

Ага, это всё. Это всё, что нам понадобится, это мир в котором мы будем творить.
Три вложенных тега — это всё, что нам нужно, чтобы начать.

Чтобы сделать нашу стартовую площадку ещё более аккуратной и чистой, давайте
добавим немного стилей, чтобы избавиться от полей и сделать фон чёрным.

    <html>
      <head>
        <style>
          body,html {
            margin:0;
            padding:0;
          }
          canvas {
            background-color: black;
          }
        </style>
      </head>
      <body>
        <canvas></canvas>
      </body>
    </html>

## Canvas-объект 

Чтобы получить доступ к canvas-объекту нам достаточно получить элементы любым удобным для нас способом.

    var canvas = document.querySelector('canvas');

Canvas-элемент может иметь несколько «контекстов» и то, что нам необходимо
сейчас это простой и обычный 2d-контекст, который позволит нам манипулировать
canvas, как растровым изображением с набором [классических методов][7].

    var ctx = canvas.getContext('2d');

Чтобы максимально увеличить область для опытов с canvas, мы можем задать приравнять размеры canvas к размерам окна.

    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;

## Цикл анимации

Цикл анимации это первая непривычная идея, с которой вы встретитесь придя из
сферы традиционной разработки приложений. При работе с графикой таким способом,
у вас обязательно возникнет желание управлять состоянием системы отдельно от
отрисовки состояния, таким образом вы придёте к двум отдельным итерациям
обновления и отрисовки. Также вам понадобится очищать холст от текущего состояния и вызывать следуюшую итерацию цикла анимации.

В итоге весь цикл занимает четыре строчки:

    function loop() {
      clear();
      update();
      draw();
      queue();
    }

Очистка холста в нашем случае — простая однострочная функция, но она станет
гораздо сложнее, если вам потребуется работаться с несколькими буферами или
состояними.

    function clear() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
    }

Вызов следующей итерации цикла мог бы быть незаурядным `setTimeout()`, но этот
способ повлечёт выполнение анимации, даже когда она не нужна пользователю
(например, он в другой вкладке), поэтому к нам помощь приходит
[requestAnimationFrame API][9], который позволит браузеру сообщать нам, есть ли
необходимость в анимировании следующего кадра или пользователь уже лайкает
котиков на ютубе. Этот метод поддерживается уже большинством браузеров, но если
вам нужна поддержка старых браузеров, то воспользуйтесь [решением Пола Айриша
(Paul Irish)][10].

    function queue() {
      window.requestAnimationFrame(loop);
    }

Методы `update()` и `draw()` вместе составляют большую часть логики, но мы можем пропустить их реализацию для того, чтобы быть увереным, что площадка для
дальнейших экспериментов готова.

    function update() {
    // пропускаем
    }
     
    function draw() {
    // пропускаем
    }
     
    loop();

Окончательный код стартовой площадки приведён ниже. Способ организации всего
оставшегося кода целиком на вашей совести, но полценный пример будет добавлен в
конце статьи для демонстрации и сравнения.

    var canvas = document.querySelector('canvas');
    var ctx = canvas.getContext('2d');
     
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
     
    function loop() {
      clear();
      update();
      draw();
      queue();
    }
     
    function clear() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
    }
     
    function update() {
    // пропускаем
    }
     
    function draw() {
    // пропускаем
    }
     
    function queue() {
      window.requestAnimationFrame(loop);
    }
     
    loop();

## Основа для частицы

На самом деле частица, это просто точка двигающаяся в двумерном пространстве. Чтобы представлять это состояние мы должны хранить, по крайней мере: координаты, скорость, и ускорение. Каждое из этих свойств может быть представлено в виде двумерного вектора, поэтому начнем с него.

### Вектор

Этот объект будет использовать **часто**. Если в анимации участвует 10 000
частик при 30 кадрах в секунду и объект вектора используется для трёх свойств
каждой частицы, то мы **создаём** около миллиона объектов каждую секунду.
Учитывая такое использование, становится очевидно, что объект вектора нуждается
в абсолютной оптимизации. В то время, как я буду оптимизировать каждый кусочек
программы, всё же из удобочитаемости и производительности, я буду выбирать
последнее. Остерегайтесь преждевременной оптимизации.

Двумерный вектор и его содержимое это обычное представление координат X и Y.

    function Vector(x, y) {
      this.x = x || 0;
      this.y = y || 0;
    }

Объект вектора должен иметь несколько методов, но на данный момент нам
необходимы немногие из них:

    // Сложить вектор с другим вектором
    Vector.prototype.add = function(vector) {
      this.x += vector.x;
      this.y += vector.y;
    }
     
    // Возвращает длину вектора
    Vector.prototype.getMagnitude = function () {
      return Math.sqrt(this.x * this.x + this.y * this.y);
    };
     
    // Возвращает угол вектора, учитывая квадрант
    Vector.prototype.getAngle = function () {
      return Math.atan2(this.y,this.x);
    };
     
    // Возвращает новый вектор, исходя из угла и размеров
    Vector.fromAngle = function (angle, magnitude) {
      return new Vector(magnitude * Math.cos(angle), magnitude * Math.sin(angle));
    };

### Частица

Из созданных нами объекта вектора, мы можем сложить объект частицы. Необходимо
передать в функцию три вектора с заданными или нулевыми координатами (`0,0`).

    function Particle(point, velocity, acceleration) {
      this.position = point || new Vector(0, 0);
      this.velocity = velocity || new Vector(0, 0);
      this.acceleration = acceleration || new Vector(0, 0);
    }

В каждом кадре необходимо смещать частицу и метод осуществляющий это прост и
незатейлив. Если частица ускоряется, надо изменить скорость, после этого
изменить координаты на значение скорости.

    Particle.prototype.move = function () {
      // Добавить ускорение к скорости
      this.velocity.add(this.acceleration);
     
      // Добавить скорость к координатам
      this.position.add(this.velocity);
    };

### Излучатель частиц

Излучатель может быть чем угодно, но в конце концов это всего-лишь точка,
излучающая частицы определённого типа. Излучателем может быть клик мыши, 
ракета, искрящийся костёр и всё это может излучать разнообразные частицы: 
пульсирующие, затухающие и даже имеющие свою собственную гравитацию.

В нашем случае излучатели просто будут выпускать частицы с заданной частотой
под определённым углом.

    function Emitter(point, velocity, spread) {
      this.position = point; // Вектор
      this.velocity = velocity; // Вектор
      this.spread = spread || Math.PI / 32; // Возможный угол = скорость +/- разброс.
      this.drawColor = "#999";
    }

Чтобы получить частицы, мы должны заставить излучатель излучать их. Это
сводится к созданию новых частиц со свойствами наследованными из свойств
излучателя и в этом случае метод `Vector.fromAngle` приходит на помощь.

    Emitter.prototype.emitParticle = function() {
      // Используйте угол, (!!!)  
      // Use an angle randomized over the spread so we have more of a "spray"
      var angle = this.velocity.getAngle() + this.spread - (Math.random() * this.spread * 2);
     
      // Магнитуда скорости излучателя
      var magnitude = this.velocity.getMagnitude();
     
      // Координаты излучателя
      var position = new Vector(this.position.x, this.position.y);
     
      // Обновлённая скорость, полученная из вычисленного угла и магнитуды
      var velocity = Vector.fromAngle(angle, magnitude);
     
      // Возвращает нашу Частицу!
      return new Particle(position,velocity);
    };

## Наша первая анимация!

Кажется у нас есть всё, чтобы начать эмулировать состояния системы частиц,
поэтому теперь мы можем приступить к реализации методов `update()` и `draw()`.

Для управления состоянием системы нам понадобятся контейнеры для частиц и
излучателей. Остановимся на простых массивах.

    var particles = [];
     
    // Добавим один излучатель с координатами `100, 230` от начала координат (верхний левый угол)
    // Начнём излучать на скорости `2` в правую сторону (угол `0`)
    var emitters = [new Emitter(new Vector(100, 230), Vector.fromAngle(0, 2))],

Как вы думаете на что должен быть похож метод `update()`? Необходимо
генерировать новые частицы и смещать их. Также было бы удобно привязать их к
определенным участкам холста, чтобы не пришлось перерисовывать весь холст.

    // Обновлённая функция update(), вызываемая из цикла анимации
    function update() {
      addNewParticles();
      plotParticles(canvas.width, canvas.height);
    }

Функция `addNewParticles()` прямолинейна — каждый излучатель генерирует набор частиц и все эти частицы добавляются в массивы частиц.

    var maxParticles = 200; // Эксперимент! 20 000 обеспечит прекрасную вселенную
    var emissionRate = 4; // количество частиц, излучаемых за кадр
     
    function addNewParticles() {
      // прекращаем, если достигнут предел
      if (particles.length > maxParticles) return;
     
      // запускаем цикл по каждому излучателю
      for (var i = 0; i < emitters.length; i++) {
     
        // согласно emissionRate, генерируем частицы
        for (var j = 0; j < emissionRate; j++) {
          particles.push(emitters[i].emitParticle());
        }
     
      }
    }

Функция `PlotParticles()` выполняется так же просто, только с небольшой
задержкой. Нам не нужно следить за частицами, улетевшими за пределы холста,
поэтому необходимо следить за этим и очищать холст для новых частиц. В этот
момент вступают в игру ограничения; скорее всего мы хотим перерисовывать не весь участок холста с частицами. В данном случае будет нормальным настроить пределы для холста.

    function plotParticles(boundsX, boundsY) {
      // Новый массив для частиц внутри холста
      var currentParticles = [];
     
      for (var i = 0; i < particles.length; i++) {
        var particle = particles[i];
        var pos = particle.position;
     
        // Если частица за пределами, то выкидываем её и переходим к следующей
        if (pos.x < 0 || pos.x > boundsX || pos.y < 0 || pos.y > boundsY) continue;
     
        // Перемещение частицы
        particle.move();
     
        // Добавление частицы в массив частиц внутри холста
        currentParticles.push(particle);
      }
     
      // Замена глобального массива частиц, на массив без вылетивших за пределы частиц
      particles = currentParticles;
    }

Состояние системы обновляется каждый кадр и теперь мы можем нарисовать что-
нибудь. В данном примере я рисую обычные квадраты, но вы можете нарисовать
искры, дым, капли воды или падающие листья. Частицы! Их больше 9 000 и они
везде!

    var particleSize = 1;
     
    function drawParticles() {
      // Задаём цвет частиц
      ctx.fillStyle = 'rgb(0,0,255)';
     
      // Запускаем цикл по частицам
      for (var i = 0; i < particles.length; i++) {
        var position = particles[i].position;
     
        // Рисуем квадрат опредлённых размеров по заданным координатам
        ctx.fillRect(position.x, position.y, particleSize, particleSize);
      }
    }

Это всё! Вы можете проверить этот пример на codepen: [демонастрация излучателя][16]

## Добавление гравитационных полей

В нашей системе, поле — это просто точка притягивающая или отталкивающая
частицы. Масса этой частицы может быть положительной (притягивает частицы) или
отрицательной (отталкивает). Я использую метод `setMass` для установки значения
массы и предлагаю различать поля по их цвету: притягивающие поля будут
зелёнными, а отталкивающие — красными.

    function Field(point, mass) {
      this.position = point;
      this.setMass(mass);
    }

    Field.prototype.setMass = function(mass) {
      this.mass = mass || 100;
      this.drawColor = mass < 0 ? "#f00" : "#0f0";
    }

Теперь мы можем создать первое в нашей системе гравитационное поле сходным с
излучателями способом.

    // Добавляем поле с координатами `400, 230` (правее излучателя)
    // установи отрицательную массу `-140`
    var fields = [new Field(new Vector(400, 230), -140)];

Нам необходимо, чтобы каждая частица обновляла свою скорость и ускорение,
основываясь на гравитационных полях рядом с собой, поэтому стоит добавить
соответствующий метод в прототип частицы. Некоторая внутренняя логика будет
перемешиваться с логикой вектора, но в данном случае она не будет вынесена в
прототип вектора, а встроена в код частицы, чтобы увеличить производительность
в этом узком месте, так как это очень дорогой метод, который будет вызываться
очень часто.

    Particle.prototype.submitToFields = function (fields) {
      // стартовое ускорение в кадре
      var totalAccelerationX = 0;
      var totalAccelerationY = 0;
     
      // запускаем цикл по гравитационным полям
      for (var i = 0; i < fields.length; i++) {
        var field = fields[i];
     
        // вычисляем расстояние между частицей и полем
        var vectorX = field.position.x - this.position.x;
        var vectorY = field.position.y - this.position.y;
     
        // вычисляем силу с помощью МАГИИ и НАУКИ!
        var force = field.mass / Math.pow(vectorX*vectorX+vectorY*vectorY,1.5);
     
        // аккумулируем ускорение в кадре произведением силы на расстояние
        totalAccelerationX += vectorX * force;
        totalAccelerationY += vectorY * force;
      }
     
      // обновляем ускорение частицы
      this.acceleration = new Vector(totalAccelerationX, totalAccelerationY);
    };

Теперь уже существующую функцию `plotParticles` надо немного обновить тем, что
добавить вызов только что реализованного метода `submitToFields` прямо перед
смещением частиц.

    function plotParticles(boundsX, boundsY) {
      var currentParticles = [];
      for (var i = 0; i < particles.length; i++) {
        var particle = particles[i];
        var pos = particle.position;
        if (pos.x < 0 || pos.x > boundsX || pos.y < 0 || pos.y > boundsY) continue;
     
        // Обновление скорости и ускорения, в соответствии с гравитацией полей
        particle.submitToFields(fields);
        
        particle.move();
        currentParticles.push(particle);
      }
      particles = currentParticles;
    }

Также было бы замечательно визуализировать поля и излучатели, поэтому давайте
добавим для этого утилитарный метод и будем вызывать его внутри функции `draw`

    // `object` это поле или излучатель (что-либо имеющее свойства drawColor и position)
    function drawCircle(object) {
      ctx.fillStyle = object.drawColor;
      ctx.beginPath();
      ctx.arc(object.position.x, object.position.y, objectSize, 0, Math.PI * 2);
      ctx.closePath();
      ctx.fill();
    }

    // Обновим функцию draw()
    function draw() {
      drawParticles();
      fields.forEach(drawCircle);
      emitters.forEach(drawCircle);
    }

## Демонстрации

Мои поздравления, вы наконец можете посмотреть финальную демку на copepen.io
[JavaScript Particle System Demo][19]

Поэкспериментируйте с разными комбинациями излучателей и гравитационных полей.
Превратите мышку в поле или излучатель с помощью синхронизации позиции курсора
и позиции излучателя или поля, на ваше усмотрение. Форкните codepen и
попробуйте что-нибудь новое!

Вы можете попробовать следующие комбинации излучателей и полей.

    var emitters = [
      new Emitter(new Vector(midX - 150, midY), Vector.fromAngle(6, 2))
    ];
     
    var fields = [
      new Field(new Vector(midX - 100, midY + 20), 150),
      new Field(new Vector(midX - 300, midY + 20), 100),
      new Field(new Vector(midX - 200, midY + 20), -20),
    ];

[Демонстрация][20]

    var emitters = [
      new Emitter(new Vector(midX - 150, midY), Vector.fromAngle(6, 2), Math.PI)
    ];
     
    var fields = [
      new Field(new Vector(midX - 300, midY + 20), 900),
      new Field(new Vector(midX - 200, midY + 10), -50),
    ];

[Демонстрация][21]

["система частиц"]: img/particle_system1-600x300.png

[1]: http://jarrodoverson.com/static/demos/particleSystem/
[2]: http://www.chromeexperiments.com/detail/gravitational-particle-system-sandbox/?f=
[4]: https://developer.mozilla.org/en-US/docs/Web/Guide/HTML/Canvas_tutorial
[5]: https://www.google.com/search?q=canvas+tutorials&oq=canvas+tutorials
[6]: https://github.com/jsoverson/html5hub-particlesystem#the-canvas-object
[7]: https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D
[9]: https://developer.mozilla.org/en-US/docs/Web/API/window.requestAnimationFrame
[10]: http://www.paulirish.com/2011/requestanimationframe-for-smart-animating/
[16]: http://cdpn.io/chGDt
[19]: http://cdpn.io/KtxmA
[20]: http://cdpn.io/pkEqs
[21]: http://cdpn.io/zyGln