
# Наблюдатели в Swift - Часть 1

Часто в разработке приложений мы оказываемся в ситуации, когда нам необходимо организовать связи типа “один-ко-многим” между объектами. Это может потребоваться, когда несколько объектов должны реагировать на изменения в одинаковом состоянии, или, когда определенные события должны транслироваться в разные части системы. 

В таких случаях часто хочется добавить некий способ наблюдения за определенными объектами. Как и с большинством техник программирования, есть несколько вариантов добавить подобное поведение к объектам в Swift, и каждый из них имеет свои сильные и слабые стороны. 

## Постановка задачи

Чтобы иметь возможность сравнить разные подходы между собой, мы должны использовать их в одинаковых условиях. Для примера мы будем использовать класс аудиоплеера, который позволяет другим объектам наблюдать за его состоянием воспроизведения. Когда плеер начинает, приостанавливает или заканчивает воспроизведение, мы хотим уведомлять наших наблюдателей, что состояние изменилось. Это позволит многим объектам связать свою логику с одним и тем же плеером - для примера, листы показывающие проигрываемые композиции, интерфейс плеера и что-то типа “мини-плеера”, показываемого сверху экрана. 

Вот как будет выглядеть наш аудиоплеер:

```
class AudioPlayer {
    private var state = State.idle {
        // Мы добавим наблюдатель к переменной 'state', который позволит нам
        // вызывать метод каждый раз, как меняется её значение.
        didSet { stateDidChange() }
    }

    func play(_ item: Item) {
        state = .playing(item)
        startPlayback(with: item)
    }

    func pause() {
        switch state {
        case .idle, .paused:
            // Вызов паузы в то время, когда плеер не играет
            // может рассматриваться как ошибка, но, тк это не 
            // несет никакого вреда, мы просто выйдем из функции тут.
            break
        case .playing(let item):
            state = .paused(item)
            pausePlayback()
        }
    }

    func stop() {
        state = .idle
        stopPlayback()
    }
}
```
Выше вы можете видеть, что мы используем технику из статьи [“Моделирование состояния в Swift”](https://www.swiftbysundell.com/posts/modelling-state-in-swift), используя enum для моделирования внутреннего состояния плеера. Это позволяет нам избавиться от опционалов и «многих источников правды», взамен предоставляя нам определенное и понятное состояние, в котором плеер может находиться. Вот как выглядит наш enum состояний плеера:

```
private extension AudioPlayer {
    enum State {
        case idle
        case playing(Item)
        case paused(Item)
    }
}
```

Выше вы могли видеть, что мы вызываем метод stateDidChange() каждый раз, когда состояние плеера изменяется. Нашей основной задачей будет - заполнить этот метод различными реализациями, в зависимости от техники, которую мы будем пробовать. 

## NotificationCenter

Первой техникой, которую мы попробуем для трансляции уведомлений об изменении состояния плеера, будет встроенный NotificationCenter. Как и многие другие API системного уровня от Apple, NotificationCenter основан на шаблоне Синглтон, но, так-как мы хотим, чтобы наш класс плеера был трестируемым (а также, чтобы было очевидно, что он зависит от NotificationCenter), мы внедрим экземпляр NotificationCenter в конструктор плеера следующим образом:

```
class AudioPlayer {
    private let notificationCenter: NotificationCenter

    init(notificationCenter: NotificationCenter = .default) {
        self.notificationCenter = notificationCenter
    }
}
```

NotificationCenter использует именованные уведомления, чтобы идентифицировать какое событие наблюдается или произошло. Чтобы избежать использования строковых переменных как части нашего API, мы добавим расширение для NotificationCenter.Name чтобы иметь единственный источник правды для имен наших уведомлений. Одно для начала воспроизведения, одно для приостановки и одно для завершения. 

```
extension Notification.Name {
    static var playbackStarted: Notification.Name {
        return .init(rawValue: "AudioPlayer.playbackStarted")
    }

    static var playbackPaused: Notification.Name {
        return .init(rawValue: "AudioPlayer.playbackPaused")
    }

    static var playbackStopped: Notification.Name {
        return .init(rawValue: "AudioPlayer.playbackStopped")
    }
}
```

_Данное расширение так же позволит пользователям нашего API легко обращаться к одному из уведомлений по имени через точку, как . playbackStarted, что всегда хорошо._

Теперь мы можем начать отправлять уведомления. Мы начнем заполнять наш метод stateDidChange() и проверять текущее состояние, чтобы определить какой тип уведомления нам следует отправлять. Для состояний проигрывания и паузы мы будем дополнительно передавать проигрываемую композицию, как объект уведомления: 

```
private extension AudioPlayer {
    func stateDidChange() {
        switch state {
        case .idle:
            notificationCenter.post(name: .playbackStopped, object: nil)
        case .playing(let item):
            notificationCenter.post(name: .playbackStarted, object: item)
        case .paused(let item):
            notificationCenter.post(name: .playbackPaused, object: item)
        }
    }
}
```

Вот и все! Теперь мы можем легко наблюдать за текущим состоянием плеера из любой части нашего кода. Для примера, вот как мы можем дать возможность наблюдать за стартом воспроизведения нашему NowPlayingViewController, для отображения текущего названия композиции и ее длительности:

```
class NowPlayingViewController: UIViewController {
    deinit {
        // Если ваше приложение поддерживает iOS 8, или более ранние версии,
        // вам необходимо вручную удалить наблюдателя. В поздних версиях
        // это делается автоматически.
        notificationCenter.removeObserver(self)
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        notificationCenter.addObserver(self,
            selector: #selector(playbackDidStart),
            name: .playbackStarted,
            object: nil
        )
    }

    @objc private func playbackDidStart(_ notification: Notification) {
        guard let item = notification.object as? AudioPlayer.Item else {
            let object = notification.object as Any
            assertionFailure("Invalid object: \(object)")
            return
        }

        titleLabel.text = item.title
        durationLabel.text = "\(item.duration)"
    }
} 
```

Основное преимущество данного подхода - это то, что его довольно легко реализовать, как для объекта, за которым осуществляется наблюдение, так и любого желающего начать наблюдать за его состоянием. Так же, большинство разработчиков знакомы с данным подход, так-как сама Apple использует его для доставки множества системных уведомлений (например, событий клавиатуры).

Однако, данный подход так же обладает рядом существенных недостатков. Во-первых, так-как NotificationCenter является Objective-C API, он не может использовать некоторые возможности Swift, такие, как дженерики, сохраняя типобезопасность. В то время, как это всегда может быть реализовано на уровне выше (с помощью некой обертки), стандартный метод использования подразумевает приведение типов, которое мы производим в первых строчках playbackDidStart() выше. Это делает наш код довольно хрупким, так-как мы не можем использовать компилятор, чтобы убедиться, что и наблюдатель, и наш объект используют одинаковый тип транслируемых значений. 

Говоря о трансляции, другой недостаток NotificationCenter это то, что уведомления транслируются на всё приложение, без какого-либо ограничения. В то время, как это может быть довольно удобно (вы можете наблюдать за любым объектом откуда угодно), это делает отношения между объектами, участвующими в наблюдении намного более свободными, что усложняет поддержание четкого разделения между разными частями приложения, особенно с ростом базы кода.  

## Протоколы наблюдателя

Далее, давайте рассмотрим, как протоколы могут быть использованы для создания более строгого и определенного API. Используя эту технику, мы будем требовать от всех объектов, заинтересованных в наблюдении за плеером, соответствовать протоколу AudioPlayerObserver. Точно так же, как мы определили три разных уведомления для каждого состояния воспроизведения, мы определим три метода, которые могут быть использованы для наблюдения за каждым событием: 

```
protocol AudioPlayerObserver: class {
    func audioPlayer(_ player: AudioPlayer,
                     didStartPlaying item: AudioPlayer.Item)

    func audioPlayer(_ player: AudioPlayer, 
                     didPausePlaybackOf item: AudioPlayer.Item)

    func audioPlayerDidStop(_ player: AudioPlayer)
}
```


Чтобы позволить выбор только одного события для наблюдения, мы дополним протокол расширением с пустыми реализациями методов для каждого события:

```
extension AudioPlayerObserver {
    func audioPlayer(_ player: AudioPlayer,
                     didStartPlaying item: AudioPlayer.Item) {}

    func audioPlayer(_ player: AudioPlayer, 
                     didPausePlaybackOf item: AudioPlayer.Item) {}

    func audioPlayerDidStop(_ player: AudioPlayer) {}
}
```

## Хранилище объектов со слабой ссылкой

При проектировании API наблюдателя это обычно хорошая практика, хранить только слабые ссылки на всех наблюдателей. В противном случае, очень легко оказаться в положении замкнутого цикла, когда владелец наблюдаемого объекта также является наблюдателем. Однако, сохранить объекты подобным образом в коллекции в Swift не всегда так просто, так-как по умолчанию все коллекции хранят сильные ссылки на свои элементы. Чтобы решить эту проблему для целей нашего наблюдателя мы введем небольшой тип-обертку который просто отслеживает наблюдателя со слабой ссылкой. 

```
private extension AudioPlayer {
    struct Observation {
        weak var observer: AudioPlayerObserver?
    }
}
```

Используя данный тип, мы теперь можем добавить коллекцию наблюдателей к нашему плееру. Для этого мы возьмем словарь с ключами типа ObjectIdentifier, чтобы получить фиксированное время добавления и удаления наблюдателей. 

```
class AudioPlayer {
    private var observations = [ObjectIdentifier : Observation]()
}
```

_ObjectIdentifier это внутренний тип, который ведет себя как уникальный идентификатор для объекта класса._

Теперь мы можем реализовать stateDidChange(), проходя по коллекции наблюдателей и вызывая метод протокола, соответствующий текущему состоянию. Стоит отметить, что в то же время мы можем воспользоваться возможностью и избавиться от неиспользуемых наблюдателей (в случае если данные объекты были освобождены).

```
private extension AudioPlayer {
    func stateDidChange() {
        for (id, observation) in observations {
            // Если наблюдатель больше не в памяти,
            // мы можем удалить его из коллекции, используя ID
            guard let observer = observation.observer else {
                observations.removeValue(forKey: id)
                continue
            }

            switch state {
            case .idle:
                observer.audioPlayerDidStop(self)
            case .playing(let item):
                observer.audioPlayer(self, didStartPlaying: item)
            case .paused(let item):
                observer.audioPlayer(self, didPausePlaybackOf: item)
            }
        }
    }
}
```

## Наблюдение

Наконец, нам нужен способ, с помощью которого объекты, соответствующие протоколу AudioPlayerObserver, могли бы регистрировать себя в качестве наблюдателей, а также, отписываться от уведомлений, в случае, если это больше не требуется. Чтобы достичь этого, мы добавим расширение для плеера, реализующее данные методы. 

```
extension AudioPlayer {
    func addObserver(_ observer: AudioPlayerObserver) {
        let id = ObjectIdentifier(observer)
        observations[id] = Observation(observer: observer)
    }

    func removeObserver(_ observer: AudioPlayerObserver) {
        let id = ObjectIdentifier(observer)
        observations.removeValue(forKey: id)
    }
}
```

Готово! Теперь мы можем обновить наш NowPlayingViewController для использования нового протокола наблюдателя, вместо NotificationCenter

```
class NowPlayingViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        player.addObserver(self)
    }
}

extension NowPlayingViewController: AudioPlayerObserver {
    func audioPlayer(_ player: AudioPlayer,
                     didStartPlaying item: AudioPlayer.Item) {
        titleLabel.text = item.title
        durationLabel.text = "\(item.duration)"
    }
}
```

Как вы можете видеть выше, основное преимущество использования явного протокола наблюдателя вместо NotificationCenter - это то, что мы имеем полную типобезопасность во время компилирования. Так-как наш протокол использует AudiPlayer.Item  напрямую, мы больше не должны прибегать к приведению типов в методе наблюдения. Как результат - более чистый и безопасный код. 

Добавление явного API наблюдателя может так же увеличить понятность того, как наблюдаемый класс работает. Вместо того, чтобы разбираться в том, как должен использоваться NotificationCenter, после взгляда на API в нашем примере, очевидно, что вы просто должны соответствовать протоколу AudioPlayerObserver, чтобы наблюдать за плеером. 

Однако, у данного подхода есть и недостаток - он требует больше кода внутри AudioPlayer, чем NotificationCenter. Он так же требует введения дополнительных протоколов и типов, что может быть недостатком, если код во многом полагается на наблюдателей. 

## Продолжение следует…

Во второй части мы продолжим разговор, рассмотрев больше подходов, которыми наблюдатели могут быть реализованы в Swift. Как всегда, я предлагаю вам поэкспериментировать с описанными техниками самостоятельно, чтобы увидеть, как они влияют на код и какой набор плюсов и минусов больше подходит для вашего проекта.  

Оригинал: https://www.swiftbysundell.com/posts/observers-in-swift-part-1
