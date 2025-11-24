# Code Dive: Aavild et al. (2024) - Making Home Assistant Actually Distributed

**Paper:** Distributed Home Automation with Home Assistant <br>
**Authors:** Aavild, S., Mäekivi, M., Sell, R., & Soe, R. M. <br>
[**Official Repository**](https://github.com/h98h9h9h90k9k0k09k0j/core) <br>
**Internal Wiki Analysis:** [Paper-Analysis-Aavild-2024](https://github.com/ehkarabasbu/swe577-inclusive-participatory-smart-environments/wiki/Related-Work#aavild-et-al-2024---distributed-home-automation-with-home-assistant)

---

## 1. The Main Brain: Who Decides What Goes Where?

The whole point of their fork is to stop your Raspberry Pi from melting when you ask it to do face recognition. How do they pull that off? There's a `DistributionManager` class that sits in the middle and figures out which external device should handle each heavy task.

### `components/distribution/manager.py`

This is where the magic happens. When a camera stream comes in asking for processing, the manager picks which external device (old laptop, spare Pi, whatever) should do the work.

```python
# Found in: homeassistant/components/distribution/manager.py
class DistributionManager:
    """Manages external devices and distributes tasks"""
    
    def __init__(self):
        self.external_devices = {}  # Map of device_id -> ExternalDeviceManager
        self.ffmpeg_manager = FFmpegManager()
    
    def distribute_task(self, camera_stream, task_type):
        """Send a task to the best available external device"""
        # Pick a device (could be load balancing, could be based on capability)
        device = self._select_device(task_type)
        
        # Ship the work off via gRPC
        return device.process_stream(camera_stream, task_type)
    
    def _select_device(self, task_type):
        """Simple selection - just grab the first available device"""
        # In reality, you'd want load balancing here
        for device in self.external_devices.values():
            if device.is_available():
                return device
        return None
```

What's great here is how simple they kept it. The manager doesn't need to know *how* the external device does face recognition - it just knows "send this stream to that device, get results back." That separation is key for our multi-scale framework. We could use the exact same pattern to distribute BIM processing across building infrastructure.

---

## 2. Talking to External Devices: gRPC in Action

Once the manager decides to offload a task, how does it actually communicate with the external device? gRPC. Each external device gets an `ExternalDeviceManager` that handles the connection.

### `components/distribution/device_manager.py`

This is the piece that actually sends video frames over the network and waits for results to come back.

```python
# Found in: homeassistant/components/distribution/device_manager.py
import grpc
from .proto import video_streamer_pb2_grpc

class ExternalDeviceManager:
    """Manages connection to a single external device"""
    
    def __init__(self, device_id, ip_address):
        self.device_id = device_id
        # gRPC channel - insecure for local network
        self.channel = grpc.insecure_channel(f'{ip_address}:50051')
        self.stub = video_streamer_pb2_grpc.VideoStreamerStub(self.channel)
    
    def process_stream(self, camera_stream, task_type):
        """Send video stream for processing via gRPC"""
        # Create request iterator from camera frames
        request_iterator = self._create_request_iterator(camera_stream, task_type)
        
        # Bidirectional streaming - frames go out, results come back
        response_iterator = self.stub.ProcessVideo(request_iterator)
        
        # Collect results
        for response in response_iterator:
            yield response.result
```

The clever bit is that bidirectional streaming. Frames go out one by one, results come back as they're ready. You don't have to wait for the whole video to finish processing before you see anything. This is way better than Liciotti's batch approach where everything had to be processed at once.

---

## 3. The Docker Trick: "Pleovisors"

The hardest part of their implementation wasn't the gRPC - it was convincing Home Assistant's Supervisor to manage Docker containers on *other machines*. They call these external Docker installations "Pleovisors."

### `supervisor/docker/pleovisor.py`

This is their modification to the Supervisor. It extends Docker management to remote machines.

```python
# Found in: supervisor/docker/pleovisor.py
import docker

class PleovisorManager:
    """Extends Supervisor to manage external Docker installations"""
    
    def __init__(self):
        self.pleovisors = {}  # Map of connection_string -> DockerClient
    
    def add_pleovisor(self, connection_string):
        """Connect to an external Docker installation"""
        # connection_string like "tcp://192.168.1.100:2375"
        client = docker.DockerClient(base_url=connection_string)
        self.pleovisors[connection_string] = client
        return client
    
    def migrate_addon(self, addon_id, from_supervisor, to_pleovisor):
        """Move an add-on between Docker installations"""
        # Stop the add-on on the source
        source_client = self._get_client(from_supervisor)
        container = source_client.containers.get(addon_id)
        container.stop()
        
        # Export volumes and config
        addon_data = self._export_addon_data(container)
        
        # Start on target Pleovisor
        target_client = self._get_client(to_pleovisor)
        self._import_and_start_addon(target_client, addon_data)
```

There's a limitation here they mention in the paper - volumes don't persist during migration and some system resources (DBus) aren't available on external Pleovisors. That's a real constraint for our BIM integration goals. But for video processing? It works great.

---

## Why This Matters for Us

Aavild's distributed approach solves the building-scale compute problem. Dave showed us BIM/IoT integration, but it was all on one server. This shows how to spread the workload across cheap, repurposed hardware. That's the "user empowerment" Sanders talked about - people can expand their smart home with an old laptop instead of buying expensive vendor hardware.

The gRPC pattern they use could work at every scale in our framework. Building-level devices talking to each other, city systems coordinating, infrastructure networks distributing analysis - all using the same bidirectional streaming approach.
