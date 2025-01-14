Что это
=======
Zactor позволяет любой руби-объект научить общаться с другими объектами посредством отправки и получения сообщений. При этом неважно где именно расположен объект, в том же процессе, в соседнем, или на другой физической машине.

Zactor использует zmq как бекенд для сообщений, eventmachine и em-zeromq для асинхронной работы с zmq, RubyInterface для описания себя, BSON для сериализации данных.

Каждый zactor-объект имеет свой identity, который генерируется автоматически, либо задается вручную и постоянен. Совокупность identity, host и port на которых был рожден этот объект, это достаточная информация для того что бы отправить этому объекту сообщение из другого объекта. Выглядит примерно так:
   
```ruby
zactor.actor => {"identity"=>"actor.2154247120-0.0.0.0:8000", "host"=>"0.0.0.0:8000"}
```

Использование
============

Для начала нужно стартануть Zactor.

```ruby
    Zactor.start 8000
```

Процесс забиндится на 0.0.0.0:8000, через эту точку будет происходить общение между zactor-процессами. Стартоваться должен в запущенном EM-контексте.

В каждый zactor-активный класс нужно делать include Zactor, после чего у класса и его экземпляров для доступа к функциями Zactor появится метод zactor. После создания объекта нужно выполнить zactor.init

```ruby
    class A
      include Zactor

      def initialize
        zactor.init
      end
    end
```

Для отправки сообщений другому объекту нам нужно знать его идентификатор. Идентификатор можно получить тремя способами:

* Непосредственной передачей. При инициализации или в любом другом месте, это исключительно внутренняя логика приложения. Идентификатор объекта можно получить вызвав zactor.actor
* При получении сообщения. В сообщении всегда содержится информация об отправителе
* Если объект имеет заранее известный identity, то мы можем получить его полный идентификатор вызвав Zactor.get_actor с identity и хостом, на котором он запущен

```ruby
    actor = Zactor.get_actor "broker", :host => "0.0.0.0:8001"
```

Получив идентификатор, можно отправлять ему сообщения

```ruby
    zactor.send_request actor, :show_me, :boobs
```

Каждый класс может определять, какие именно события он может получать и что с ними делать

```ruby
    include Zactor

    zactor do
      event(:show_me) do |o, msg, what|
        case what
        when :boobs
          do_it
        else
          do_smth_else
        end
      end
    end
```

Рассмотрим пример банального ping-pong

```ruby
    class A
      include Zactor

      def initialize
        zactor.init
        ping Zactor.get_actor("b")
      end

      def ping(actor) 
        puts "Ping!"
        zactor.send_request actor, :ping do |res|
          puts res
        end
      end
    end

    class B
      include Zactor

      zactor do
        identity "b"
    
        event(:ping) do |o, msg|
          msg.reply "Pong!"
        end
      end
  
      def initialize
        zactor.init
      end

    end

    EM.run do
      Zactor.start 8000
  
      a = A.new
      b = B.new
    end
```

A посылает сообщение :ping для B, а B отвечает "Pong!"

В коллбэк, определенный в event, передается объект, получивший сообщение, объект сообщения ({Zactor::Message}) и далее переданные в запросе аргументы (если они есть). У {Zactor::Message} есть два основных метода: sender, возвращающий идентификатор отправителя и reply, который посылает ответ на запрос.

Важный момент, identity должно задаваться ДО zactor.init и после этого не может меняться.

ZMQ
===

При Zactor.start стартует брокер, по одному на каждый процесс, через него проходят все сообщения данного процесса, принимает сообщения через SUB-сокет, отправляет через PUB. SUB подписан на все сообщения. Каждый zactor-объект создает по паре сокетов, PUB подключается к SUB-брокера, а SUB к PUB-брокера. SUB подписывается на сообщения содержащие его identity.

![ZMQ](https://github.com/Undev/zactor/raw/master/images/zactor1.png)

Рассмотрим жизнь сообщения на примере с ping-ping. В случае с b в том же процессе:

<div class=wsd wsd_style="default"><pre>
A[PUB]->Broker[SUB]: Посылаем запрос :ping
Broker[SUB]->Broker[PUB]: Перебрасываем запрос в PUB сокет
Broker[PUB]->B[SUB]: Передаем получателю сообщение
B[PUB]->Broker[SUB]: Отправляем ответ "Pong!"
Broker[SUB]->Broker[PUB]: Перебрасываем запрос в PUB сокет
Broker[PUB]->A[SUB]: Отправитель получает ответ
</pre></div><script type="text/javascript" src="http://www.websequencediagrams.com/service.js"></script>

В случае с b в другом процессе:

<div class=wsd wsd_style="default"><pre>
A[PUB for App2]->App2 Broker[SUB]: Посылаем запрос :ping
App2 Broker[SUB]->App2 Broker[PUB]: Перебрасываем запрос в PUB сокет
App2 Broker[PUB]->B[SUB]: Передаем получателю сообщение
B[PUB for App1]->App1 Broker[SUB]: Отправляем ответ "Pong!"
App1 Broker[SUB]->App1 Broker[PUB]: Перебрасываем запрос в PUB сокет
App1 Broker[PUB]->A[SUB]: Отправитель получает ответ
</pre></div><script type="text/javascript" src="http://www.websequencediagrams.com/service.js"></script>

Балансировка
============

Так как это ZMQ, мы можем очень просто изменить тип получения сообщения. Например, добавив балансер. Теперь можно запускать процессы со ссылкой на этот балансер.

    Zactor.start :balancer => "0.0.0.0:4000"
    
У нас получится примерно следующая схема:

![ZMQ](https://github.com/Undev/zactor/raw/master/images/zactor2.png)

Теперь наш ping можно отправлять в балансер, а отвечать будет один из подключенных воркеров.

    ping Zactor.get_actor("b", :host => "0.0.0.0:4000")
    

Протокол обмена
===============


Perfomance
==========

А хрен его знает, толком не мерялось ничего :)

TODO
====

* Сделать событие отваливания объектов. Наверное, что-то вроде простого аналога link в эрланге.
* Добавить таймауты для запросов с коллбэками. Сейчас они будут висеть бесконечно и засрут память.
* Доступ к отправителю в колллбэке запроса. В случае с балансировкой он будет не тем же, кому мы посылали сообщение