
# Наблюдатели в Swift - Часть 2

В прошлой статье мы рассмотрели разные пути реализации шаблона наблюдатель в Swift: NotificationCenter API и протоколы наблюдателя, организовать наблюдение за аудиоплеером. Во второй части мы сконцентрируемся на нескольких техниках, основанных на замыканиях.

## Замыкания 

Замыкания играют значительную роль в проектировании современных API. Начиная с асинхронных API с обратными вызовами (callbacks), до функциональных операций (например, использование forEach или map в коллекциях); замыкания везде - как в стандартной библиотеке Swift, так и в приложениях, которые мы разрабатываем. 

Прелесть замыканий в том, что они позволяют пользователю API захватывать текущий контекст, в котором они находятся и затем использовать его, реагируя на события. В нашем случае - на изменение состояния плеера. Это упрощает вызов метода, что, в свою очередь, упрощает код. Однако, замыкания так же имеют свои недостатки. 

Давайте попробуем добавить основанное на замыканиях API для нашего плеера. Мы начнем с объявления кортежа (множества), который мы будем использовать для отслеживания массивов наблюдающих замыканий, которые мы добавим для каждого из трех наших событий (плей, пауза, стоп). 

```
class AudioPlayer {
    private var observations = (
        started: [(AudioPlayer, Item) -> Void](),
        paused: [(AudioPlayer, Item) -> Void](),
        stopped: [(AudioPlayer) -> Void]()
    )
}
```

_Конечно, мы можем объявить каждое из замыканий в отдельном свойстве, но использование кортежей для группировки связанных, свойств может оказаться действительно удобным для простой и удобной организации вещей._

Теперь давайте определим наши методы наблюдения. Так же, как и в предыдущих подходах, мы определим отдельный метод для каждого события, сохраняя четкое разделение. Каждый метод принимает замыкание. Так же мы будем передавать проигрываемую в настоящий момент композицию в наблюдатели статуса play и pause, в то время, как в наблюдатель stop мы передадим только сам плеер.

```
extension AudioPlayer {
    func observePlaybackStarted(using closure: @escaping (AudioPlayer, Item) -> Void) {
        observations.started.append(closure)
    }

    func observePlaybackPaused(using closure: @escaping (AudioPlayer, Item) -> Void) {
        observations.paused.append(closure)
    }

    func observePlaybackStopped(using closure: @escaping (AudioPlayer) -> Void) {
        observations.stopped.append(closure)
    }
}
```

_Передача самого плеера во все замыкания - это действительно хорошая практика, помогающая избежать случайных замкнутых циклов, в случае, когда плеер используется внутри одного из своих наблюдающих замыканий и мы забыли сделать ссылку на него “слабой”._

Теперь, все что остается, это вызвать наблюдающие замыкания для соответствующих событий, когда состояние плеера изменяется:

```
private extension AudioPlayer {
    func stateDidChange() {
        switch state {
        case .idle:
            observations.stopped.forEach { closure in
                closure(self)
            }
        case .playing(let item):
            observations.started.forEach { closure in
                closure(self, item)
            }
        case .paused(let item):
            observations.paused.forEach { closure in
                closure(self, item)
            }
        }
    }
}
```

Давайте испытаем наше новое API. Вот как наш NowPlayingViewController будет выглядеть с использованием данного подхода:

```
class NowPlayingViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()

        let titleLabel = self.titleLabel
        let durationLabel = self.durationLabel

        player.observePlaybackStarted { player, item in
            titleLabel.text = item.title
            durationLabel.text = "\(item.duration)"
        }
    }
}
```

Однако, есть один большой недостаток данного подхода - нет способа удалить наблюдателя (отписаться от наблюдения). В то время, как это не является проблемой для владельцев наблюдаемых объектов (так-как они будут освобождены вместе с их владельцем), это не очень удобно для общих объектов (типа нашего аудиоплеера) так-как добавленный наблюдатель будет жить вечно.

## Токены

Один из возможных путей избавиться от наблюдателей - использование токенов. Мы можем возвращать ObservationToken каждый раз, когда наблюдатель добавлен, что может быть впоследствии использовано для отмены наблюдения и удаления наблюдающего замыкания. 

Давайте начнем с создания класса токена, выполняющего роль обертки, которая может быть вызвана для отмены наблюдения:

```
class ObservationToken {
    private let cancellationClosure: () -> Void

    init(cancellationClosure: @escaping () -> Void) {
        self.cancellationClosure = cancellationClosure
    }

    func cancel() {
        cancellationClosure()
    }
}
```

Так как замыкания невозможно идентифицировать, нам нужен какой-то способ точно опознать наблюдателя, чтобы мы могли его удалить. Один из способов добиться этого - это просто превратить наш массив замыканий в словарь с UUID-значениями в качестве ключей:

```
class AudioPlayer {
    private var observations = (
        started: [UUID : (AudioPlayer, Item) -> Void](),
        paused: [UUID : (AudioPlayer, Item) -> Void](),
        stopped: [UUID : (AudioPlayer) -> Void]()
    )
}
```

Теперь, назначим UUID для каждого замыкания в момент добавления. Чтобы упростить процесс, мы можем добавить простое расширение для словаря, позволяющее нам создавать и добавлять идентификатор на ходу:

```
private extension Dictionary where Key == UUID {
    mutating func insert(_ value: Value) -> UUID {
        let id = UUID()
        self[id] = value
        return id
    }
}
```

Теперь мы можем обновить наши методы наблюдения так, чтобы они возвращали токены. Мы используем @discardableResult, чтобы сделать реальное использование возвращаемых токенов необязательным, чтобы избежать предупреждений в случае наблюдения за объектами, которыми владеет сам наблюдатель. 

В каждом методе мы будем использовать наш метод insert и, затем, создавать экземпляр ObservationToken с замыканием, которое удаляет наблюдателя: 

```
extension AudioPlayer {
    @discardableResult
    func observePlaybackStarted(using closure: @escaping (AudioPlayer, Item) -> Void)
        -> ObservationToken {
        let id = observations.started.insert(closure)

        return ObservationToken { [weak self] in
            self?.observations.started.removeValue(forKey: id)
        }
    }

    @discardableResult
    func observePlaybackPaused(using closure: @escaping (AudioPlayer, Item) -> Void)
        -> ObservationToken {
        let id = observations.paused.insert(closure)

        return ObservationToken { [weak self] in
            self?.observations.paused.removeValue(forKey: id)
        }
    }

    @discardableResult
    func observePlaybackStopped(using closure: @escaping (AudioPlayer) -> Void)
        -> ObservationToken {
        let id = observations.stopped.insert(closure)

        return ObservationToken { [weak self] in
            self?.observations.stopped.removeValue(forKey: id)
        }
    }
}
```

Прежде чем мы попробуем наше новое токен-API в действии, мы должны также изменить метод stateDidChange() в нашем плеере так, чтобы мы могли использовать свойства values в процессе прохождения по словарю (так-как ключи нас не интересуют в данной ситуации):

```
private extension AudioPlayer {
    func stateDidChange() {
        switch state {
        case .idle:
            observations.stopped.values.forEach { closure in
                closure(self)
            }
        case .playing(let item):
            observations.started.values.forEach { closure in
                closure(self, item)
            }
        case .paused(let item):
            observations.paused.values.forEach { closure in
                closure(self, item)
            }
        }
    }
}    
```

Теперь наш NowPlayingViewController может отписываться от уведомлений в процессе деинициализации:

```
class NowPlayingViewController: UIViewController {
    private var observationToken: ObservationToken?

    deinit {
        observationToken?.cancel()
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        let titleLabel = self.titleLabel
        let durationLabel = self.durationLabel

        observationToken = player.observePlaybackStarted { player, item in  
            titleLabel.text = item.title
            durationLabel.text = "\(item.duration)"
        }
    }
}
```

В то время, как API основанное на токенах может быть полезным в ситуациях, когда вам необходимо динамически добавлять и удалять наблюдателей, для нескольких объектов, как вы можете видеть выше, это не самый простой путь, в случае если все что мы хотим, это просто добавить наблюдателя. Забыть вызвать cancel для токена так же может стать частой ошибкой, что требует от нас хранить больше состояния в наших наблюдателях (сохраняя токены).

Давайте посмотрим, как мы можем улучшить это решение!

## Лучшее из двух миров

Давайте рассмотрим способ сохранить наше основанное на токенах API (так-как оно позволяет нам более гибко управлять нашими наблюдателями) и избавиться от шаблонного кода. 

Один из способов сделать это - связать наблюдающее замыкание с жизненным циклом самого наблюдателя без необходимости наблюдателя следовать какому-либо протоколу. Все что мы сделаем - это добавим API, которое позволит нам передавать любой объект как наблюдателя вместе с замыканием, как и раньше. Далее, мы захватим слабую ссылку на этот объект и будем проверять, не является ли он nil, чтобы определить актуально ли еще наблюдение:

```
extension AudioPlayer {
    @discardableResult
    func addPlaybackStartedObserver<T: AnyObject>(
        _ observer: T,
        closure: @escaping (T, AudioPlayer, Item) -> Void
    ) -> ObservationToken {
        let id = UUID()

        observations.started[id] = { [weak self, weak observer] player, item in
            // Если наблюдатель был уничтожен, мы автоматически
            // удаляем наблюдающее замыкание.
            guard let observer = observer else {
                self?.observations.started.removeValue(forKey: id)
                return
            }

            closure(observer, player, item)
        }

        return ObservationToken { [weak self] in
            self?.observations.started.removeValue(forKey: id)
        }
    }
}
```

Таким образом мы берем лучшее от обоих подходов. Мы можем использовать токены, когда нам это необходимо, а также автоматически удалять любых наблюдателей, которых добавил объект, при его деинициализации. Теперь мы можем безопасно наблюдать за нашим плеером в NowPlayingViewController:

```
class NowPlayingViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()

        player.addPlaybackStartedObserver(self) {
            vc, player, item in

            vc.titleLabel.text = item.title
            vc.durationLabel.text = "\(item.duration)"
        }
    }
}
```

Выше вы можете наблюдать реальную пользу от передачи самого наблюдателя, даже в подходе с замыканиями, так-как передача наблюдателя вместе с замыканием избавляет нас от необходимости вручную захватывать self или любое нужное нам свойство.

_Оригинал: https://www.swiftbysundell.com/posts/observers-in-swift-part-2_
