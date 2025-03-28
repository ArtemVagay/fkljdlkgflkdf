import socket
import pygame
import threading

pygame.init()

# Настройки окна
WIDTH, HEIGHT = 800, 600
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Multiplayer Game")

# Цвета
WHITE = (255, 255, 255)
RED = (255, 0, 0)
BLUE = (0, 0, 255)

# Сетевые настройки
client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.connect(('localhost', 5555))

player_pos = {"x": 50, "y": 50}
other_players = {}


def receive_updates():
    while True:
        try:
            data = client.recv(1024).decode('utf-8')
            if data.startswith("update:"):
                _, player_id, x, y = data.split(":")
                other_players[int(player_id)] = {"x": int(x), "y": int(y)}
        except:
            print("Disconnected from server")
            client.close()
            break


# Запускаем поток для получения обновлений
receive_thread = threading.Thread(target=receive_updates)
receive_thread.daemon = True
receive_thread.start()

# Игровой цикл
running = True
clock = pygame.time.Clock()

while running:
    clock.tick(60)
    screen.fill(WHITE)

    # Обработка событий
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False

    # Движение игрока
    keys = pygame.key.get_pressed()
    if keys[pygame.K_LEFT]:
        player_pos["x"] -= 5
    if keys[pygame.K_RIGHT]:
        player_pos["x"] += 5
    if keys[pygame.K_UP]:
        player_pos["y"] -= 5
    if keys[pygame.K_DOWN]:
        player_pos["y"] += 5

    # Отправляем позицию на сервер
    client.send(f"move:{player_pos['x']}:{player_pos['y']}".encode('utf-8'))

    # Рисуем игрока
    pygame.draw.rect(screen, RED, (player_pos["x"], player_pos["y"], 50, 50))

    # Рисуем других игроков
    for player_id, pos in other_players.items():
        pygame.draw.rect(screen, BLUE, (pos["x"], pos["y"], 50, 50))

    pygame.display.flip()

pygame.quit()
client.close()
