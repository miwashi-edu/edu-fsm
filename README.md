# edu-fsm

## On/Off

```python
from machine import Pin, PWM
import time

print("Starting FSM")
# ---------- FSM core ----------
class State:
    def enter(self, fsm): pass
    def exit(self, fsm): pass
    def on_event(self, fsm, event): pass


class FSM:
    def __init__(self, initial_state):
        self.state = initial_state
        self.state.enter(self)

    def dispatch(self, event):
        new_state = self.state.on_event(self, event)
        if new_state:
            self.state.exit(self)
            self.state = new_state
            self.state.enter(self)


# ---------- LED1: toggle ----------
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
    def __init__(self, pin):
        self.led = Pin(pin, Pin.OUT)
        super().__init__(Off())


# ---------- Setup ----------
button_pins = [15]

fsms = [
    ToggleFSM(16),
]

pin_to_fsm = {}
last_press = {}
event_queue = []


# ---------- Interrupt ----------
def make_handler(pin_num):
    def handler(pin):
        print(f"Installing handler for pin ${pin}")
        now = time.ticks_ms()
        if time.ticks_diff(now, last_press[pin_num]) > 200:
            event_queue.append(pin_num)
            last_press[pin_num] = now
            print(now)
    return handler


for btn, fsm in zip(button_pins, fsms):
    p = Pin(btn, Pin.IN, Pin.PULL_UP)
    print(f"pinning {btn} {fsm}")
    pin_to_fsm[btn] = fsm
    last_press[btn] = 0
    p.irq(trigger=Pin.IRQ_FALLING, handler=make_handler(btn))

# ---------- Main loop ----------
while True:
    while event_queue:
        pin = event_queue.pop(0)
        pin_to_fsm[pin].dispatch("press")

    # tick all FSMs
    for fsm in fsms:
        fsm.dispatch("tick")

    time.sleep_ms(50)
```

