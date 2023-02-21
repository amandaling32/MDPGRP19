import sys
import time
import ast
from typing import List

import settings
from app import AlgoSimulator, AlgoMinimal
from entities.assets.direction import Direction
from entities.connection.rpi_client import RPiClient
from entities.connection.rpi_server import RPiServer
from entities.grid.obstacle import Obstacle


def parse_obstacle_data(data) -> List[Obstacle]:
    obs = []
    for obstacle_params in data:
        obs.append(Obstacle(obstacle_params[0],
                            obstacle_params[1],
                            Direction(obstacle_params[2]),
                            obstacle_params[3]))
    # [[x, y, orient, index], [x, y, orient, index]]
    return obs


def run_simulator():
    # Fill in obstacle positions with respect to lower bottom left corner.
    # (x-coordinate, y-coordinate, Direction)
    obstacles = [[135, 25, 0, 1], [55, 75, -90, 2], [195, 95, 180, 3], [175, 185, -90, 4], [75, 125, 90, 5], [15, 185, -90, 6]]
   

    
    obs = parse_obstacle_data(obstacles)
    app = AlgoSimulator(obs)
    app.init()
    commands = "C"+str(app.robot.convert_all_commands())
    print(commands)
    app.execute()
    


def run_minimal(also_run_simulator):
    # Create a client to connect to the RPi.
    print(f"Attempting to connect to {settings.RPI_HOST}:{settings.RPI_PORT}")
    client = RPiClient(settings.RPI_HOST, settings.RPI_PORT)
    # Wait to connect to RPi.
    while True:
        try:
            client.connect()
            break
        except OSError:
            pass
        except KeyboardInterrupt:
            client.close()
            sys.exit(1)
    
    print("Connected to RPi!\n")
    while True:
        payload = client.socket.recv(1024).decode()
        print("Waiting to receive obstacle data from RPi...")

        print("Got data from RPi:")
        obstacle_data = ast.literal_eval(payload)
        print(obstacle_data)
        
        obstacles = parse_obstacle_data(obstacle_data)
        if also_run_simulator:
            app = AlgoSimulator(obstacles)
            app.init()
            app.execute()
        
        app = AlgoMinimal(obstacles)
        app.init()
        app.execute()
        # #
        # # # Send the list of commands over.
        print("Sending list of commands to RPi...")
        commands = "C"+str(app.robot.convert_all_commands())
        #commands = app.robot.convert_all_commands()
        client.send_message(commands)
        print(commands)
   

def run_rpi():
    while True:
        run_minimal(False)
        time.sleep(5)


if __name__ == '__main__':
    #run_rpi()
    run_simulator()