import pygame
import random
import time
import os
import json

pygame.init()
pygame.mixer.init()

ANCHO, ALTO = 600, 600
pantalla = pygame.display.set_mode((ANCHO, ALTO))
pygame.display.set_caption("Juego de Esquivar Chuzos")

# Colores
NEGRO = (0, 0, 0)
AMARILLO = (255, 255, 0)
VERDE = (0, 255, 0)
BLANCO = (255, 255, 255)
ROJO = (255, 0, 0)

fuente = pygame.font.SysFont(None, 30)
reloj = pygame.time.Clock()

monedas_recolectadas = 0
record_guardado = 0
pastillas = 0
logros = {}

# Logros iniciales
logros_definicion = {
    "primer_moneda": {"desc": "Recolecta tu primera moneda", "completado": False},
    "cazador_oro": {"desc": "Recolecta 50 monedas", "completado": False},
    "superviviente": {"desc": "Sobrevive 60 segundos", "completado": False},
    "invencible": {"desc": "Sobrevive 180 segundos", "completado": False},
    "destructor": {"desc": "Destruye 50 chuzos", "completado": False}
}

chuzos_destruidos_total = 0

# ==== Cargar sonidos ====
sonido_moneda = pygame.mixer.Sound("sonidos/moneda.wav")
sonido_disparo = pygame.mixer.Sound("sonidos/disparo.wav")
sonido_danio = pygame.mixer.Sound("sonidos/danio.wav")
sonido_gameover = pygame.mixer.Sound("sonidos/gameover.wav")

pygame.mixer.music.load("sonidos/musica_fondo.mp3")
pygame.mixer.music.set_volume(0.5)
pygame.mixer.music.play(-1)

# Cargar imágenes
fondo = pygame.image.load("imagenes/fondo.png")
fondo = pygame.transform.scale(fondo, (ANCHO, ALTO))

img_jugador = pygame.image.load("imagenes/jugador.png")
img_jugador = pygame.transform.scale(img_jugador, (40, 40))

img_chuzo = pygame.image.load("imagenes/chuzo.png")
img_chuzo = pygame.transform.scale(img_chuzo, (20, 40))

img_moneda = pygame.image.load("imagenes/moneda.png")
img_moneda = pygame.transform.scale(img_moneda, (20, 20))

img_vida = pygame.image.load("imagenes/vida.png")
img_vida = pygame.transform.scale(img_vida, (25, 25))

img_disparo = pygame.image.load("imagenes/disparo.png")
img_disparo = pygame.transform.scale(img_disparo, (15, 20))

mensajes_educativos = [
    "El VIH es el Virus de Inmunodeficiencia Humana.",
    "El VIH debilita las defensas del cuerpo al atacar el sistema inmunológico.",
    "Infórmate, cuídate y respeta a quienes viven con VIH.",
    "El VIH no se transmite por abrazos ni besos.",
    "Usar preservativo previene el VIH.",
    "Hacerse la prueba es un acto de responsabilidad.",
    "Tener VIH no impide llevar una vida plena.",
    "La educación es la mejor herramienta contra el VIH.",
    "El tratamiento adecuado permite controlar el VIH.",
    "No discrimines a quienes viven con VIH."
]

def cargar_record():
    if os.path.exists("record.txt"):
        with open("record.txt", "r") as archivo:
            return int(archivo.read())
    return 0

def guardar_record(nuevo_record):
    with open("record.txt", "w") as archivo:
        archivo.write(str(nuevo_record))

def guardar_progreso():
    progreso = {
        "monedas": monedas_recolectadas,
        "resurgir": resurgir,
        "escudo": escudo,
        "aliados_comprados": aliados_comprados,
        "pastillas": pastillas,
        "logros": logros,
        "chuzos_destruidos_total": chuzos_destruidos_total
    }
    with open("progreso.json", "w") as archivo:
        json.dump(progreso, archivo)

def cargar_progreso():
    global monedas_recolectadas, resurgir, escudo, aliados_comprados, pastillas, logros, chuzos_destruidos_total
    if os.path.exists("progreso.json"):
        with open("progreso.json", "r") as archivo:
            progreso = json.load(archivo)
            monedas_recolectadas = progreso.get("monedas", 0)
            resurgir = progreso.get("resurgir", False)
            escudo = progreso.get("escudo", 0)
            aliados_comprados = progreso.get("aliados_comprados", 0)
            pastillas = progreso.get("pastillas", 0)
            logros = progreso.get("logros", logros_definicion.copy())
            chuzos_destruidos_total = progreso.get("chuzos_destruidos_total", 0)
    else:
        logros.update(logros_definicion)

def desbloquear_logro(nombre):
    global monedas_recolectadas, resurgir
    if not logros[nombre]["completado"]:
        logros[nombre]["completado"] = True
        mostrar_mensaje_logro(f"¡Logro desbloqueado! {logros[nombre]['desc']}")
        if all(l["completado"] for l in logros.values()):
            monedas_recolectadas += 50
            resurgir = True
            mostrar_mensaje_logro("¡Todos los logros completados! +50 monedas y Resurgir gratis")

def mostrar_mensaje_logro(texto):
    tiempo_inicio = time.time()
    while time.time() - tiempo_inicio < 2:
        pantalla.blit(fondo, (0, 0))
        mostrar_texto(texto, 50, 300, AMARILLO)
        pygame.display.flip()
        for evento in pygame.event.get():
            if evento.type == pygame.QUIT:
                pygame.quit()
                exit()

class Jugador(pygame.sprite.Sprite):
    def __init__(self):
        super().__init__()
        self.image = img_jugador
        self.rect = self.image.get_rect()
        self.rect.centerx = ANCHO // 2
        self.rect.bottom = ALTO - 20
        self.velocidad = 5

    def update(self):
        teclas = pygame.key.get_pressed()
        if teclas[pygame.K_LEFT]:
            self.rect.x -= self.velocidad
        if teclas[pygame.K_RIGHT]:
            self.rect.x += self.velocidad
        if self.rect.right < 0:
            self.rect.left = ANCHO
        elif self.rect.left > ANCHO:
            self.rect.right = 0

class ObjetoCae(pygame.sprite.Sprite):
    def __init__(self, image, width, height, velocidad):
        super().__init__()
        self.image = image
        self.rect = self.image.get_rect()
        self.rect.x = random.randint(0, ANCHO - width)
        self.rect.y = random.randint(-600, -40)
        self.velocidad = velocidad

    def update(self):
        self.rect.y += self.velocidad
        if self.rect.top > ALTO:
            self.kill()

class Chuzo(ObjetoCae):
    def __init__(self):
        super().__init__(img_chuzo, 20, 40, random.randint(4, 8))

class Moneda(ObjetoCae):
    def __init__(self):
        super().__init__(img_moneda, 20, 20, 5)

class VidaExtra(ObjetoCae):
    def __init__(self):
        super().__init__(img_vida, 25, 25, 4)

class Disparo(pygame.sprite.Sprite):
    def __init__(self, x, y):
        super().__init__()
        self.image = img_disparo
        self.rect = self.image.get_rect()
        self.rect.centerx = x
        self.rect.bottom = y
        self.velocidad = -10

    def update(self):
        self.rect.y += self.velocidad
        if self.rect.bottom < 0:
            self.kill()

class BotAliado(pygame.sprite.Sprite):
    def __init__(self, grupo_disparos, grupo_todos):
        super().__init__()
        self.image = pygame.transform.scale(img_jugador, (30, 30))
        self.rect = self.image.get_rect()
        self.rect.x = random.randint(0, ANCHO - 30)
        self.rect.y = random.randint(300, 500)
        self.velocidad_x = random.choice([-3, 3])
        self.tiempo_disparo = time.time()
        self.grupo_disparos = grupo_disparos
        self.grupo_todos = grupo_todos

    def update(self):
        self.rect.x += self.velocidad_x
        if self.rect.left <= 0 or self.rect.right >= ANCHO:
            self.velocidad_x *= -1
        if time.time() - self.tiempo_disparo > 1:
            disparo = Disparo(self.rect.centerx, self.rect.top)
            self.grupo_disparos.add(disparo)
            self.grupo_todos.add(disparo)
            self.tiempo_disparo = time.time()

def mostrar_texto(texto, x, y, color=BLANCO):
    imagen = fuente.render(texto, True, color)
    pantalla.blit(imagen, (x, y))

def pantalla_game_over():
    sonido_gameover.play()
    mensajes = random.sample(mensajes_educativos, 3)
    while True:
        pantalla.blit(fondo, (0, 0))
        mostrar_texto("GAME OVER", 220, 150, ROJO)
        mostrar_texto(mensajes[0], 50, 220)
        mostrar_texto(mensajes[1], 50, 260)
        mostrar_texto(mensajes[2], 50, 300)
        mostrar_texto("Presiona M para volver al Menú", 150, 380, AMARILLO)
        pygame.display.flip()
        for evento in pygame.event.get():
            if evento.type == pygame.QUIT:
                guardar_progreso()
                pygame.quit()
                exit()
            if evento.type == pygame.KEYDOWN:
                if evento.key == pygame.K_m:
                    guardar_progreso()
                    return

def ver_logros():
    while True:
        pantalla.blit(fondo, (0, 0))
        mostrar_texto("LOGROS", 250, 50, AMARILLO)
        y = 120
        for nombre, datos in logros.items():
            estado = "✔" if datos["completado"] else "✘"
            mostrar_texto(f"{estado} {datos['desc']}", 50, y, VERDE if datos["completado"] else ROJO)
            y += 40
        mostrar_texto("Presiona M para volver al menú", 100, 500, BLANCO)
        pygame.display.flip()
        for evento in pygame.event.get():
            if evento.type == pygame.QUIT:
                guardar_progreso()
                pygame.quit()
                exit()
            if evento.type == pygame.KEYDOWN:
                if evento.key == pygame.K_m:
                    return
def menu_principal():
    global monedas_recolectadas
    while True:
        pantalla.blit(fondo, (0, 0))
        mostrar_texto("MENÚ PRINCIPAL", 200, 150)
        mostrar_texto("1 - Iniciar Juego", 200, 220)
        mostrar_texto("2 - Tienda", 200, 260)
        mostrar_texto("3 - Ver Logros", 200, 300)
        mostrar_texto(f"Monedas: {monedas_recolectadas}", 200, 360, AMARILLO)
        pygame.display.flip()
        for evento in pygame.event.get():
            if evento.type == pygame.QUIT:
                guardar_progreso()
                pygame.quit()
                exit()
            if evento.type == pygame.KEYDOWN:
                if evento.key == pygame.K_1:
                    guardar_progreso()
                    return
                elif evento.key == pygame.K_2:
                    guardar_progreso()
                    tienda()
                elif evento.key == pygame.K_3:
                    ver_logros()

def tienda():
    global monedas_recolectadas, escudo, resurgir, aliados_comprados, pastillas
    while True:
        pantalla.blit(fondo, (0, 0))
        mostrar_texto("TIENDA", 250, 100)
        mostrar_texto("1 - Resurgir (10 monedas)", 150, 160)
        mostrar_texto("2 - Pastilla de curación (2 monedas)", 150, 200)
        mostrar_texto("3 - Escudo (3 monedas)", 150, 240)
        mostrar_texto("4 - Aliado (5 monedas)", 150, 280)
        mostrar_texto("5 - Volver al Menú", 150, 340)
        mostrar_texto(f"Monedas: {monedas_recolectadas}", 150, 380, AMARILLO)
        mostrar_texto(f"Aliados comprados: {aliados_comprados}/2", 150, 420, VERDE)
        mostrar_texto(f"Pastillas: {pastillas}", 150, 460, BLANCO)
        pygame.display.flip()
        for evento in pygame.event.get():
            if evento.type == pygame.QUIT:
                guardar_progreso()
                pygame.quit()
                exit()
            if evento.type == pygame.KEYDOWN:
                if evento.key == pygame.K_1 and monedas_recolectadas >= 10:
                    resurgir = True
                    monedas_recolectadas -= 10
                elif evento.key == pygame.K_2 and monedas_recolectadas >= 2:
                    pastillas += 1
                    monedas_recolectadas -= 2
                elif evento.key == pygame.K_3 and monedas_recolectadas >= 3:
                    escudo = 5
                    monedas_recolectadas -= 3
                elif evento.key == pygame.K_4 and monedas_recolectadas >= 5 and aliados_comprados < 2:
                    aliados_comprados += 1
                    monedas_recolectadas -= 5
                elif evento.key == pygame.K_5:
                    guardar_progreso()
                    return

def jugar():
    global monedas_recolectadas, record_guardado, resurgir, escudo, aliados_comprados, pastillas, chuzos_destruidos_total
    todos = pygame.sprite.Group()
    chuzos = pygame.sprite.Group()
    monedas = pygame.sprite.Group()
    vidas_extras = pygame.sprite.Group()
    disparos = pygame.sprite.Group()
    disparos_bot = pygame.sprite.Group()
    bots = pygame.sprite.Group()

    jugador = Jugador()
    todos.add(jugador)

    for _ in range(aliados_comprados):
        bot = BotAliado(disparos_bot, todos)
        bots.add(bot)
        todos.add(bot)

    vida_maxima = 100
    vida_actual = vida_maxima
    inicio_tiempo = time.time()
    tiempo_moneda = 0
    tiempo_vida = 0

    for _ in range(10):
        chuzo = Chuzo()
        chuzos.add(chuzo)
        todos.add(chuzo)

    ejecutando = True
    while ejecutando:
        reloj.tick(60)
        tiempo_actual = time.time()
        tiempo_transcurrido = int(tiempo_actual - inicio_tiempo)

        for evento in pygame.event.get():
            if evento.type == pygame.QUIT:
                guardar_progreso()
                ejecutando = False
            if evento.type == pygame.KEYDOWN:
                if evento.key == pygame.K_SPACE:
                    disparo = Disparo(jugador.rect.centerx, jugador.rect.top)
                    disparos.add(disparo)
                    todos.add(disparo)
                    sonido_disparo.play()
                elif evento.key == pygame.K_h and pastillas > 0 and vida_actual < vida_maxima:
                    vida_actual = min(vida_actual + int(vida_maxima * 0.15), vida_maxima)
                    pastillas -= 1

        if tiempo_actual - tiempo_moneda > 5:
            moneda = Moneda()
            monedas.add(moneda)
            todos.add(moneda)
            tiempo_moneda = tiempo_actual

        if tiempo_actual - tiempo_vida > 20:
            vida = VidaExtra()
            vidas_extras.add(vida)
            todos.add(vida)
            tiempo_vida = tiempo_actual

        if len(chuzos) < 10:
            nuevo_chuzo = Chuzo()
            chuzos.add(nuevo_chuzo)
            todos.add(nuevo_chuzo)

        todos.update()
        disparos_bot.update()

        if tiempo_transcurrido >= 60:
            desbloquear_logro("superviviente")
        if tiempo_transcurrido >= 180:
            desbloquear_logro("invencible")

        if pygame.sprite.spritecollide(jugador, chuzos, True):
            if escudo > 0:
                escudo -= 1
            else:
                vida_actual -= 25
                sonido_danio.play()
            if vida_actual <= 0:
                if resurgir:
                    vida_actual = vida_maxima
                    resurgir = False
                else:
                    if tiempo_transcurrido > record_guardado:
                        guardar_record(tiempo_transcurrido)
                    guardar_progreso()
                    ejecutando = False
                    pantalla_game_over()

        for disparo in disparos:
            chuzos_golpeados = pygame.sprite.spritecollide(disparo, chuzos, True)
            if chuzos_golpeados:
                disparo.kill()
                chuzos_destruidos_total += len(chuzos_golpeados)
                if chuzos_destruidos_total >= 50:
                    desbloquear_logro("destructor")
                for _ in chuzos_golpeados:
                    nuevo_chuzo = Chuzo()
                    chuzos.add(nuevo_chuzo)
                    todos.add(nuevo_chuzo)

        for disparo in disparos_bot:
            chuzos_golpeados = pygame.sprite.spritecollide(disparo, chuzos, True)
            if chuzos_golpeados:
                disparo.kill()
                chuzos_destruidos_total += len(chuzos_golpeados)
                if chuzos_destruidos_total >= 50:
                    desbloquear_logro("destructor")
                for _ in chuzos_golpeados:
                    nuevo_chuzo = Chuzo()
                    chuzos.add(nuevo_chuzo)
                    todos.add(nuevo_chuzo)

        for moneda in pygame.sprite.spritecollide(jugador, monedas, True):
            monedas_recolectadas += 1
            sonido_moneda.play()
            if monedas_recolectadas >= 1:
                desbloquear_logro("primer_moneda")
            if monedas_recolectadas >= 50:
                desbloquear_logro("cazador_oro")

        for vida in pygame.sprite.spritecollide(jugador, vidas_extras, True):
            vida_actual = min(vida_actual + 25, vida_maxima)

        pantalla.blit(fondo, (0, 0))
        todos.draw(pantalla)
        mostrar_texto(f"Tiempo: {tiempo_transcurrido}s", 10, 10)
        dibujar_barra_vida(10, 40, vida_actual, vida_maxima)
        mostrar_texto(f"Récord: {max(record_guardado, tiempo_transcurrido)}s", 10, 70)
        mostrar_texto(f"Monedas: {monedas_recolectadas}", 10, 100, AMARILLO)
        if escudo > 0:
            mostrar_texto(f"Escudo: {escudo}", 10, 130, VERDE)
        mostrar_texto(f"Pastillas: {pastillas}", 10, 160, BLANCO)
        mostrar_texto("Presiona H para curarte", 10, 190, VERDE)
        pygame.display.flip()

    aliados_comprados = 0
    guardar_progreso()

def dibujar_barra_vida(x, y, vida_actual, vida_maxima):
    ancho_barra = 100
    alto_barra = 15
    proporcion = vida_actual / vida_maxima
    largo_vida = int(ancho_barra * proporcion)
    pygame.draw.rect(pantalla, ROJO, (x, y, ancho_barra, alto_barra))
    pygame.draw.rect(pantalla, VERDE, (x, y, largo_vida, alto_barra))
    pygame.draw.rect(pantalla, BLANCO, (x, y, ancho_barra, alto_barra), 2)

record_guardado = cargar_record()
cargar_progreso()

while True:
    menu_principal()
    jugar()
