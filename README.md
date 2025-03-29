import pygame
import socket
import json
import threading

# Initialize pygame
pygame.init()
WIDTH, HEIGHT = 800, 600
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Multiplayer Game")
clock = pygame.time.Clock()

# Colors
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
RED = (255, 0, 0)
GREEN = (0, 255, 0)
BLUE = (0, 0, 255)


class GameClient:
    def __init__(self, host='localhost', port=5555):
        self.client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.host = host
        self.port = port
        self.player_id = None
        self.game_state = {
            'players': {},
            'bullets': [],
            'scores': {}
        }
        self.keys = {
            'up': False,
            'down': False,
            'left': False,
            'right': False
        }
        self.player_speed = 5
        self.bullet_speed = 10

    def connect(self):
        try:
            self.client.connect((self.host, self.port))
            receive_thread = threading.Thread(target=self.receive)
            receive_thread.daemon = True
            receive_thread.start()
            return True
        except:
            return False

    def send(self, data):
        try:
            self.client.send(json.dumps(data).encode('utf-8'))
        except:
            print("Connection lost")

    def receive(self):
        while True:
            try:
                data = json.loads(self.client.recv(1024).decode('utf-8'))

                if data['type'] == 'init':
                    self.player_id = data['player_id']
                    self.game_state = data['game_state']

                elif data['type'] == 'update':
                    self.game_state = data['game_state']

            except:
                print("Disconnected from server")
                self.client.close()
                break

    def move_player(self):
        if self.player_id is None or self.player_id not in self.game_state['players']:
            return

        player = self.game_state['players'][self.player_id]
        dx, dy = 0, 0

        if self.keys['up']:
            dy -= self.player_speed
        if self.keys['down']:
            dy += self.player_speed
        if self.keys['left']:
            dx -= self.player_speed
        if self.keys['right']:
            dx += self.player_speed

        new_x = player['x'] + dx
        new_y = player['y'] + dy

        # Keep player within screen bounds
        new_x = max(20, min(WIDTH - 20, new_x))
        new_y = max(20, min(HEIGHT - 20, new_y))

        if new_x != player['x'] or new_y != player['y']:
            self.send({
                'type': 'move',
                'x': new_x,
                'y': new_y
            })

    def shoot(self, target_x, target_y):
        if self.player_id is None or self.player_id not in self.game_state['players']:
            return

        player = self.game_state['players'][self.player_id]
        dx = target_x - player['x']
        dy = target_y - player['y']

        # Normalize direction
        length = (dx ** 2 + dy ** 2) ** 0.5
        if length > 0:
            dx = dx / length * self.bullet_speed
            dy = dy / length * self.bullet_speed

            self.send({
                'type': 'shoot',
                'x': player['x'],
                'y': player['y'],
                'dx': dx,
                'dy': dy
            })


def draw_game(screen, game_state, player_id):
    screen.fill(BLACK)

    # Draw players
    for pid, player in game_state['players'].items():
        color = player.get('color', RED) if pid != player_id else GREEN
        pygame.draw.circle(screen, color, (int(player['x']), int(player['y'])), 20)

    # Draw bullets
    for bullet in game_state['bullets']:
        pygame.draw.circle(screen, WHITE, (int(bullet['x']), int(bullet['y'])), 5)

    # Draw scores
    font = pygame.font.SysFont(None, 36)
    y_pos = 10
    for pid, score in game_state['scores'].items():
        text = f"Player {pid}: {score}"
        if pid == player_id:
            text += " (YOU)"
        text_surface = font.render(text, True, WHITE)
        screen.blit(text_surface, (10, y_pos))
        y_pos += 40

    pygame.display.flip()


def main():
    client = GameClient()  # Change to server IP if needed
    if not client.connect():
        print("Failed to connect to server")
        return

    running = True
    while running:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False

            elif event.type == pygame.KEYDOWN:
                if event.key == pygame.K_w:
                    client.keys['up'] = True
                if event.key == pygame.K_s:
                    client.keys['down'] = True
                if event.key == pygame.K_a:
                    client.keys['left'] = True
                if event.key == pygame.K_d:
                    client.keys['right'] = True

            elif event.type == pygame.KEYUP:
                if event.key == pygame.K_w:
                    client.keys['up'] = False
                if event.key == pygame.K_s:
                    client.keys['down'] = False
                if event.key == pygame.K_a:
                    client.keys['left'] = False
                if event.key == pygame.K_d:
                    client.keys['right'] = False

            elif event.type == pygame.MOUSEBUTTONDOWN:
                if event.button == 1:  # Left mouse button
                    mouse_x, mouse_y = pygame.mouse.get_pos()
                    client.shoot(mouse_x, mouse_y)

        client.move_player()
        draw_game(screen, client.game_state, client.player_id)
        clock.tick(60)

    pygame.quit()


if __name__ == "__main__":
    main()
