import socket
import threading

clients = []
positions = {}

def handle_client(client_socket, addr):
    print(f"[NEW CONNECTION] {addr} connected.")
    player_id = len(clients) - 1
    positions[player_id] = {"x": 50, "y": 50}
    
    while True:
        try:
            data = client_socket.recv(1024).decode('utf-8')
            if not data:
                break
                
            # Обработка движения: формат "move:x:y"
            if data.startswith("move:"):
                _, x, y = data.split(":")
                positions[player_id]["x"] = int(x)
                positions[player_id]["y"] = int(y)
                
                # Отправляем обновленные позиции всем клиентам
                update_msg = f"update:{player_id}:{x}:{y}"
                for client in clients:
                    client.send(update_msg.encode('utf-8'))
                    
        except ConnectionResetError:
            break
            
    print(f"[DISCONNECTED] {addr} disconnected")
    client_socket.close()
    clients.remove(client_socket)
    del positions[player_id]

def start_server():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.bind(('0.0.0.0', 5555))
    server.listen()
    print("[SERVER] Server is listening on port 5555")
    
    while True:
        client_socket, addr = server.accept()
        clients.append(client_socket)
        thread = threading.Thread(target=handle_client, args=(client_socket, addr))
        thread.start()
        print(f"[ACTIVE CONNECTIONS] {threading.active_count() - 1}")

if __name__ == "__main__":
    start_server()
