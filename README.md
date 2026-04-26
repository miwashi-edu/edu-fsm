# edu-fsm

## Project file tree

```text
fsm_project/
│
├── main.py
├── fsm.py
│
├── states/
│   ├── __init__.py
│   ├── toggle.py
│
└── README.md
```



## main.py

```python
from machine import Pin
import time

from states.toggle import ToggleFSM

button_pins = [15]
led_pins = [16]

pin_to_fsm = {}
event_queue = []
last_press = {}


def make_handler(pin_num):
    def handler(pin):
        now = time.ticks_ms()
        if time.ticks_diff(now, last_press[pin_num]) > 200:
            event_queue.append(pin_num)
            last_press[pin_num] = now
    return handler


for btn, led in zip(button_pins, led_pins):

    fsm = ToggleFSM(led, Pin)

    pin_to_fsm[btn] = fsm
    last_press[btn] = 0

    p = Pin(btn, Pin.IN, Pin.PULL_UP)
    p.irq(trigger=Pin.IRQ_FALLING, handler=make_handler(btn))


while True:
    while event_queue:
        pin_to_fsm[event_queue.pop(0)].dispatch("press")

    time.sleep_ms(20)
```

## fsm.py

```python
class State:
    def enter(self, fsm):
        pass

    def exit(self, fsm):
        pass

    def on_event(self, fsm, event):
        return None


class FSM:
    def __init__(self, initial_state):
        self.state = initial_state
        self.state.enter(self)

    def dispatch(self, event):
        new_state = self.state.on_event(self, event)

        if new_state and new_state != self.state:
            self.state.exit(self)
            self.state = new_state
            self.state.enter(self)
```

## states/toggle.py

```python
from fsm import FSM, State


class Off(State):
    def enter(self, fsm):
        fsm.led.value(0)

    def on_event(self, fsm, event):
        if event == "press":
            return On()


class On(State):
    def enter(self, fsm):
        fsm.led.value(1)

    def on_event(self, fsm, event):
        if event == "press":
            return Off()


class ToggleFSM(FSM):
    def __init__(self, led_pin, Pin):
        self.led = Pin(led_pin, Pin.OUT)
        self.led.value(0)
        super().__init__(Off())
```

## states/blink.py

```python
from fsm import FSM, State
from machine import Pin
import time


class Off(State):
    def enter(self, fsm):
        fsm.led.value(0)

    def on_event(self, fsm, event):
        return On()


class On(State):
    def enter(self, fsm):
        fsm.led.value(1)

    def on_event(self, fsm, event):
        return Off()


class BlinkFSM(FSM):
    def __init__(self, pin, PinClass, interval_ms=500):
        self.led = PinClass(pin, Pin.OUT)
        self.interval = interval_ms
        self.last = time.ticks_ms()
        super().__init__(Off())

    def tick(self):
        now = time.ticks_ms()
        if time.ticks_diff(now, self.last) > self.interval:
            self.last = now
            self.dispatch("toggle")
```

## states/dim.py

```python
from fsm import FSM, State
from machine import Pin, PWM


class DimState(State):
    def enter(self, fsm):
        fsm.brightness = 30000
        fsm.pwm.duty_u16(fsm.brightness)

    def on_event(self, fsm, event):
        if event == "up":
            fsm.brightness = min(65535, fsm.brightness + 5000)
        elif event == "down":
            fsm.brightness = max(0, fsm.brightness - 5000)

        fsm.pwm.duty_u16(fsm.brightness)
        return self


class DimFSM(FSM):
    def __init__(self, pin, PinClass):
        self.pwm = PWM(PinClass(pin))
        self.pwm.freq(1000)
        super().__init__(DimState())
```

## states/fade.py

```python
from fsm import FSM, State
from machine import Pin, PWM
import time


class FadeState(State):
    def enter(self, fsm):
        fsm.direction = 1
        fsm.value = 0

    def on_event(self, fsm, event):
        return self

    def update(self, fsm):
        fsm.value += fsm.direction * 1000

        if fsm.value >= 65535:
            fsm.value = 65535
            fsm.direction = -1

        if fsm.value <= 0:
            fsm.value = 0
            fsm.direction = 1

        fsm.pwm.duty_u16(int(fsm.value))


class FadeFSM(FSM):
    def __init__(self, pin, PinClass):
        self.pwm = PWM(PinClass(pin))
        self.pwm.freq(1000)
        super().__init__(FadeState())

    def tick(self):
        self.state.update(self)
```



