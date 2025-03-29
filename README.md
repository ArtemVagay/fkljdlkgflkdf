import socket
import threading
import json


class GameServer:
    def __init__(self, host='0.0.0.0', port=5555):
        self.host = host
        self.port = port
        self.server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server.bind((host, port))
        self.server.listen()
        self.clients = []
        self.players = {}
        self.game_state = {
            'players': {},
            'bullets': [],
            'scores': {}
        }

    def broadcast(self, data):
        for client in self.clients:
            try:
                client.send(json.dumps(data).encode('utf-8'))
            except:
                self.clients.remove(client)

    def handle_client(self, client, addr):
        print(f"[NEW CONNECTION] {addr} connected.")

        # Assign player ID
        player_id = str(addr[1])
        self.players[player_id] = {'x': 100, 'y': 100, 'color': (255, 0, 0)}
        self.game_state['players'][player_id] = self.players[player_id]
        self.game_state['scores'][player_id] = 0

        # Send initial game state
        client.send(json.dumps({
            'type': 'init',
            'player_id': player_id,
            'game_state': self.game_state
        }).encode('utf-8'))

        while True:
            try:
                data = json.loads(client.recv(1024).decode('utf-8'))

                if data['type'] == 'move':
                    self.players[player_id]['x'] = data['x']
                    self.players[player_id]['y'] = data['y']
                    self.game_state['players'] = self.players
                    self.broadcast({
                        'type': 'update',
                        'game_state': self.game_state
                    })

                elif data['type'] == 'shoot':
                    bullet = {
                        'x': data['x'],
                        'y': data['y'],
                        'dx': data['dx'],
                        'dy': data['dy'],
                        'owner': player_id
                    }
                    self.game_state['bullets'].append(bullet)
                    self.broadcast({
                        'type': 'update',
                        'game_state': self.game_state
                    })

                elif data['type'] == 'hit':
                    self.game_state['scores'][player_id] += 1
                    self.broadcast({
                        'type': 'update',
                        'game_state': self.game_state
                    })

            except:
                print(f"[DISCONNECTED] {addr} disconnected.")
                del self.players[player_id]
                del self.game_state['players'][player_id]
                del self.game_state['scores'][player_id]
                self.clients.remove(client)
                client.close()
                self.broadcast({
                    'type': 'update',
                    'game_state': self.game_state
                })
                break

    def update_game(self):
        # Update bullets positions
        for bullet in self.game_state['bullets'][:]:
            bullet['x'] += bullet['dx']
            bullet['y'] += bullet['dy']

            # Remove bullets that are out of bounds
            if (bullet['x'] < 0 or bullet['x'] > 800 or
                    bullet['y'] < 0 or bullet['y'] > 600):
                self.game_state['bullets'].remove(bullet)

        # Check for collisions
        for bullet in self.game_state['bullets'][:]:
            for player_id, player in self.game_state['players'].items():
                if player_id != bullet['owner']:
                    if (abs(bullet['x'] - player['x']) < 20 and
                            abs(bullet['y'] - player['y']) < 20):
                        self.game_state['bullets'].remove(bullet)
                        self.game_state['scores'][bullet['owner']] += 1
                        break

        self.broadcast({
            'type': 'update',
            'game_state': self.game_state
        })

    def run(self):
        print(f"[SERVER STARTED] on {self.host}:{self.port}")

        # Start game update thread
        update_thread = threading.Thread(target=self._update_loop)
        update_thread.daemon = True
        update_thread.start()

        # Accept connections
        while True:
            client, addr = self.server.accept()
            self.clients.append(client)
            thread = threading.Thread(target=self.handle_client, args=(client, addr))
            thread.daemon = True
            thread.start()

    def _update_loop(self):
        import time
        while True:
            self.update_game()
            time.sleep(0.016)  # ~60 FPS


if __name__ == "__main__":
    server = GameServer()
    server.run()
