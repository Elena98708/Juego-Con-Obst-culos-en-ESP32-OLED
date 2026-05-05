# Juego-Con-Obst-culos-en-ESP32-OLED
from machine import Pin, I2C, PWM, Timer
from ssd1306 import SSD1306_I2C
import time, random

# ==========================
# CONFIGURACIÓN
# ==========================
SCL_PIN = 22
SDA_PIN = 21

BTN_UP, BTN_DOWN, BTN_START = 14, 27, 12
BUZZER_PIN = 13

# ==========================
# MODOS
# ==========================
MODOS = [
    ("Clasico",       (3, 800)),
    ("Contra-tiempo", (3, 700)),
    ("Hardcore",      (5, 400))
]

DURACION_CONTRATIEMPO = 60000

# ==========================
# OLED
# ==========================
i2c = I2C(0, scl=Pin(SCL_PIN), sda=Pin(SDA_PIN))
oled = SSD1306_I2C(128, 64, i2c)

btn_up = Pin(BTN_UP, Pin.IN, Pin.PULL_UP)
btn_down = Pin(BTN_DOWN, Pin.IN, Pin.PULL_UP)
btn_start = Pin(BTN_START, Pin.IN, Pin.PULL_UP)

buzzer = PWM(Pin(BUZZER_PIN))
buzzer.deinit()

# ==========================
# SPRITES
# ==========================
SPRITE_RABBIT = [
"01011010","01011010","00111100","01111110",
"11111111","10111101","01000010","00111100"
]

SPRITE_RABBIT_2 = [
"01011010","00111100","01111110","11111111",
"01111110","00111100","01011010","00000000"
]

SPRITE_OBS = [
"00111000","01111100","11111110","11111110",
"11111110","01111100","00111000","00000000"
]

SPRITE_BOOM = [
"10000001","01011010","00111100","11111111",
"00111100","01011010","10000001","00000000"
]

# ==========================
# FUNCIONES
# ==========================
def dibujar_sprite(x, y, sprite):
    for f in range(8):
        for c in range(8):
            if sprite[f][c] == "1":
                oled.pixel(x + c, y + f, 1)

def beep(freq, dur):
    buzzer.init(freq=freq, duty=512)
    time.sleep_ms(dur)
    buzzer.deinit()

# ==========================
# VARIABLES
# ==========================
estado, modo = "MENU", 0
jugador_y, puntaje, obstaculos = 28, 0, []
esquivados = 0

inicio_tiempo = 0
tiempo_pausa = 0

# Animación
frame = 0
ultimo_frame = 0

# Dificultad
velocidad, spawn_rate = 3, 800

# ==========================
# FUNCIONES JUEGO
# ==========================
def reset():
    global jugador_y, puntaje, obstaculos, esquivados
    global inicio_tiempo, velocidad, spawn_rate

    jugador_y = 28
    puntaje = 0
    obstaculos = []
    esquivados = 0

    inicio_tiempo = time.ticks_ms()

    _, (velocidad, spawn_rate) = MODOS[modo]

def animacion():
    global frame, ultimo_frame
    if time.ticks_diff(time.ticks_ms(), ultimo_frame) > 150:
        frame = 1 - frame
        ultimo_frame = time.ticks_ms()

def actualizar():
    global obstaculos, puntaje, esquivados, velocidad, spawn_rate

    # Spawn
    if random.randint(0, 100) < 5:
        obstaculos.append([120, random.randint(0, 56)])

    nuevos = []
    for obs in obstaculos:
        obs[0] -= velocidad
        if obs[0] > 0:
            nuevos.append(obs)
        else:
            puntaje += 5
            esquivados += 1

    obstaculos = nuevos
    puntaje += 1

    # Escalado dificultad (solo clásico)
    if modo == 0 and puntaje % 200 == 0:
        velocidad += 1
        spawn_rate = max(300, spawn_rate - 50)

def colision():
    for obs in obstaculos:
        if 10 < obs[0]+8 and 18 > obs[0] and jugador_y < obs[1]+8 and jugador_y+8 > obs[1]:
            return True
    return False

# ==========================
# DIBUJO
# ==========================
def dibujar():
    oled.fill(0)

    if estado == "MENU":
        oled.text("DODGER", 30, 0)
        for i,(n,_) in enumerate(MODOS):
            oled.text((">" if i==modo else " ")+n, 10, 20+i*10)

    elif estado in ["JUEGO","PAUSA"]:

        # Conejo animado
        if frame == 0:
            dibujar_sprite(10, jugador_y, SPRITE_RABBIT)
        else:
            dibujar_sprite(10, jugador_y, SPRITE_RABBIT_2)

        for obs in obstaculos:
            dibujar_sprite(obs[0], obs[1], SPRITE_OBS)

        oled.text(f"P:{puntaje}", 80, 0)
        oled.text(f"E:{esquivados}", 0, 0)
        oled.text(MODOS[modo][0][:4], 45, 0)

        # Contra-tiempo
        if modo == 1:
            restante = max(0, DURACION_CONTRATIEMPO - time.ticks_diff(time.ticks_ms(), inicio_tiempo))
            oled.text(f"T:{restante//1000}", 45, 10)

        if estado == "PAUSA":
            oled.text("PAUSA", 40, 30)

    elif estado == "GAME_OVER":
        oled.text("GAME OVER", 20, 10)
        oled.text(f"P:{puntaje}", 30, 30)
        oled.text(f"E:{esquivados}", 30, 45)

    oled.show()

# ==========================
# LOOP
# ==========================
while True:

    if estado == "MENU":
        if not btn_up.value():
            modo = (modo-1)%3
            beep(1000,50)
            time.sleep_ms(200)

        elif not btn_down.value():
            modo = (modo+1)%3
            beep(800,50)
            time.sleep_ms(200)

        elif not btn_start.value():
            reset()
            estado = "JUEGO"
            beep(1200,100)
            time.sleep_ms(300)

    elif estado == "JUEGO":

        animacion()

        if not btn_start.value():
            estado = "PAUSA"
            tiempo_pausa = time.ticks_ms()
            time.sleep_ms(200)

        if not btn_up.value() and jugador_y>0:
            jugador_y -= 4

        if not btn_down.value() and jugador_y<56:
            jugador_y += 4

        actualizar()

        # Fin contra-tiempo
        if modo == 1:
            if time.ticks_diff(time.ticks_ms(), inicio_tiempo) >= DURACION_CONTRATIEMPO:
                estado = "GAME_OVER"

        if colision():
            dibujar_sprite(10, jugador_y, SPRITE_BOOM)
            oled.show()
            beep(300,300)
            time.sleep(1)
            estado = "GAME_OVER"

    elif estado == "PAUSA":
        if not btn_start.value():
            inicio_tiempo += time.ticks_diff(time.ticks_ms(), tiempo_pausa)
            estado = "JUEGO"
            time.sleep_ms(200)

    elif estado == "GAME_OVER":
        time.sleep(2)
        estado = "MENU"

    dibujar()
    time.sleep_ms(50)
