# widowx_envs

Code for controlling Trossen WidowX robot arms.

### Project structure

- `widowx_envs`: contains the `widowx_envs` Python package with all of the WidowX controller code.
- `docker_compose.yml`: contains all of the docker-compose services that will be used to run the robot.

### Host machine setup
In order install dependencies on the host machine, run `host_install.sh`.

### Setup
First, we need to build and launch the ROS service that communicates with the robot. This service is defined by the `robonet` entry in `docker-compose.yml`. It uses the `robonet-base` image which is built from the `widowx_envs` directory (`see widowx_envs/Dockerfile`). To build and run the `robonet-base` service, run:

```
USB_CONNECTOR_CHART=/absolute/path/to/usb_connector_chart.yml docker compose up --build robonet
```

This builds the `robonet-base` image, which contains all of the ROS dependencies and the Python controller code from the `widowx_envs` directory. The USB connector chart is required to start the camera stream. You can get the USB device IDs by running `v4l2-ctl --list-devices`.

Once this is running, you can execute commands in the running container like so:

```
docker compose exec robonet bash -lic "go_sleep"
```

Explanation:
- `docker compose exec`: execute a command in a running container
- `robonet`: the service name (as specified in `docker-compose.yml`)
- `bash`: the executable to run inside the container
- `-l`: tells bash to open a login shell, sourcing `~/.bashrc` inside the container, which is required to set up a few ROS things and the correct Python virtual environment
- `-i`: makes the shell interactive, in case you want to run an interactive command (like `python`)
- `-c`: tells bash to execute the next argument
- `go_sleep`: the string to be executed by bash; in this case, it's a utility script that is built in to the `robonet-base` image that moves the arm to the sleep position

If you really want to, you can also attach a bash shell interactively using `docker compose exec robonet bash`.

### Adding Codebases
Additional codebases are incorporated by adding additional services to `docker-compose.yml`. See `docker-compose.yml` for an example with the `bridge_data_v2` codebase. Each codebase also needs a minimal Dockerfile that builds on top of the `robonet-base` image. An example Dockerfile might look like:

```Dockerfile
FROM robonet-base:latest

RUN ~/myenv/bin/pip install tensorflow

# this path will get mounted as a volume (see `docker-compose.yml`)
ENV PYTHONPATH=${PYTHONPATH}:/home/robonet/code/bridge_data_v2

WORKDIR /home/robonet/code/bridge_data_v2
```

First, we build the new image:

```
docker compose build bridge_data_v2
```

Now, we can run commands in the container similar to the previous section, except we don't even need to have the container running in the background all the time!

```
docker compose run bridge_data_v2 bash -lic "python experiments/something/eval_policy.py ..."
```

Instead of executing a command in a running container, `docker compose run` spins up the container, executes the command, and then shuts down the container when the command finishes. Spinning the container up and down adds very little overhead. However, if you really want an interactive shell, you can again do `docker compose run bridge_data_v2 bash`.


### Using RealSense cameras

The RealSense cameras require different drivers than RGB cameras.  If you are using RealSenses, change the `camera_string` in `scripts/run.sh` to `realsense:=true`.

You will also need to update the device IDs in `widowx_envs/widowx/launch/realsense.launch` to match your cameras.

## Troubleshooting

##### Permission errors

If you run into following errors:

```
Traceback (most recent call last):
  File "urllib3/connectionpool.py", line 677, in urlopen
  File "urllib3/connectionpool.py", line 392, in _make_request
  File "http/client.py", line 1277, in request
  File "http/client.py", line 1323, in _send_request
  File "http/client.py", line 1272, in endheaders
  File "http/client.py", line 1032, in _send_output
  File "http/client.py", line 972, in send
  File "docker/transport/unixconn.py", line 43, in connect
PermissionError: [Errno 13] Permission denied
```
that can be fixed by running the following commands and subsequently restarting the PC (the log out and log back in is sometimes not sufficient):

```
sudo groupadd docker
sudo usermod -aG docker $USER
```
