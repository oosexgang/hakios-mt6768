# userland/camera/

The iOS side of `/dev/redwood-camera` (kernel: `../../kernel/bsd/dev/redwood_mtkcam.c`, boot-arg `mtkcam=5`):

| file | what |
| --- | --- |
| `camera_device.c` | the capture device iOS's CMCapture graph is given, inside `mediaserverd` (inserted by the common user space's `libredwood-media`): streams, the processing session, buffer pools, the frames of the device read as RAW10 and made into pictures |
| `camera_isp.c`, `camera_isp.h` | what the phone's ISP would do, done by the processor: levels, AE, white balance, lens shading, a colour matrix, a tone curve, demosaic, scaling, NV12 and BGRA |

They build with the common user space's `userland/build.sh` (they need `../../kernel/bsd/sys/redwood_camera.h` on the include path); this repository has no user-space build of its own.
Debug aids are off by default: `CAM_TRACE` (the Objective-C message trace of CMCapture; it also needs `-DCAM_TRACE_STUB=0x...`, the unslid address of an `objc_stubs` entry of CMCapture in the iOS cache you built from: no address of any cache is kept in this tree), `CAM_MARK` (marks the pictures).
How it was found, run by run: `../../docs/camera-x6826b.md`. The home-screen icon of Camera.app needs one more piece in SpringBoard's own library
(the visible icon tags lack `still-camera`; see the same document).
