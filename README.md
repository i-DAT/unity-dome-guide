# Fulldome for Unity 6

This guide explains how to set up a Unity 6 project for use on a fulldome, using the repositories in the i-DAT organization.

This guide has been tested on Unity `6000.1.6f1`.

## Project Setup

1. Create a blank Unity project. The `Universal 3D` template is recommended.
2. Go to `Edit > Project Settings > Player` and set the following options:
    - `Other Settings > API Compatibility Level` set to `.NET Framework`
    - `Resolution and Presentation > Fullscreen Mode` set to `Windowed`
    - `Resolution and Presentation > Default Screen Width / Height` set to a square, eg `2048x2048`
3. Create a `Scripts` folder.

## Camera Setup

1. Add the [fulldome camera](https://github.com/i-DAT/unity-fulldome-camera) repository to the `Scripts/` folder.

   This can be achieved by opening the `Scripts/` folder in a command prompt and running `git clone https://github.com/i-DAT/unity-fulldome-camera`.
   Alternatively, download the source code from Github through `Code > Download ZIP`, extract it, and place contents into `Scripts/`.

2. Navigate into `Scripts/unity-fulldome-camera/Runtime`.
   Add the `FisheyeCam` script to your main camera, and drag the `Fisheye` shader into the shader field of the script.

3. Disabe `Rendering > Post Processing` on the camera component. 

## NDI Setup

1. Add the [NDI](https://github.com/keijiro/KlakNDI) repository to the `Scripts` folder using the same method as above.

2. Navigate into `Scripts/KlakNDI`. It is important to remove the `HDRP`, `URP`, and `Legacy` folders as they will cause compilation errors.

3. Navigate into `jp.keijiro.klak.ndi/Runtime/Component`. Add the `NdiSender` script to your main camera and ensure that `Capture method` is set to `Game View`.
   
   The Unity game view should now appear in NDI receiver programs as `<host nane> (Game View)`.
   For testing, the [DistroAV](https://github.com/DistroAV/DistroAV) OBS plugin can be used to view NDI streams on a host computer.

4. You may need to enter play mode once to ensure that NDI is running.

## OSC Setup

1. Add the [OSC](https://github.com/i-DAT/unity-osc) repositorepository to the `Scripts` folder using the same method as above.

2. Install the [sensor app](https://github.com/i-DAT/sensor-app) on a device to send OSC data to Unity.

3. Create or modify a script for your player controller. It must implement the `IOscClient` interface.
 
   When an OSC message is received from a client, a method is invoked with the name of the OSC address.
   If this is not found, a default method is called.

   Here is an example client implementation:
    ```csharp
    using System.Collections.Concurrent;
    using System.Net;

    public class Player : MonoBehaviour, IOscClient
    {
        // Required to implement the interface, only useful for sending OSC packets back to the client.
        public IPEndPoint EndPoint { get; set; }
        public ConcurrentQueue<OscPacket> SendQueue { get; set; }

        Vector3 angle;

        // Set the player rotation to the phone rotation each frame.
        void Update() => transform.eulerAngles = angle;

        // Called each time an OSC message with the `/rotation` address is sent from the phone.
        public void rotation(float x, float y, float z) => angle = new Vector3(x, y, z) * Mathf.Rad2Deg;

        // Required to implement the interface.
        public void OnMessage(OscMessage m) => Debug.Log($"Unhandled OSC message {m}!");
    }
    ```

4. Create a new script (such as `Manager`) and add it to an object in your scene (such as a new `Empty` or to the main camera).
   It must store an `OscManager` object and create new client objects when a phone connects.

   Its important to call the `manager.Update()` each frame and call `manager.Dispose` when the object dies.

   Here is an example client manager implementation:
    ```csharp
    public class Manager : MonoBehaviour
    {
        OscManager manager = new();

        void Start()
        {
            // Set the `OnConnect` callback to create a client each time a phone connects.
            manager.OnConnect = (endpoint) => {
                var player = new Player();
                // Only necessary for sending OSC packets back to the client.
                player.Endpoint = endpoint;
                player.SendQueue = manager.sendQueue;

                return player;
            };
            // Start the manager to listen for OSC messages.
            manager.Start();
        }

        void Update() => manager.Update();

        void OnDestroy() => manager.Dispose();
    }
    ```

    More complete demos can be found in [unity-fulldome-multiplayer-demo](https://github.com/i-DAT/unity-fulldome-multiplayer-demo) and [flight-simulator](https://github.com/i-DAT/flight-simulator).