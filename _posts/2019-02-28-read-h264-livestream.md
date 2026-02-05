---

title: Capturing H.264 Livestreams from IP Cameras

category: Gist
tags: [Geek, Python, FFmpeg, OpenCV]
date: 2019-02-28
---

I recently acquired a solar-powered WiFi IP camera from eBay: [Wireless Solar IP67 Security Camera System Outdoor Home Cam 1080P 2MP](https://www.ebay.com.au/itm/302918525683?ul_noapp=true).

My goal was to capture video or images from the camera programmatically using a script.

### Using FFmpeg

You can use `ffmpeg` to capture the stream directly:

```bash
ffmpeg -re -i "http://host_ip:81/livestream.cgi?user=admin&pwd=&streamid=0" -c copy -f mpegts test.mp4
```

To capture a single frame as an image:

```bash
ffmpeg -i "http://host_ip:81/livestream.cgi?user=admin&pwd=a123&streamid=0" -c copy -f mpegts -ss 5 -frames:v 5 testt.png
```

### Using OpenCV (Python)

If you need to process the stream in Python, `cv2` (OpenCV) is a great alternative, especially since creating a portable FFmpeg wrapper can be difficult.

(Reference: [Not able to play .h264 video on OpenCV?](https://stackoverflow.com/questions/28477600/not-able-to-play-h264-video-on-opencv))

```python
import cv2

# Replace with your actual camera URL
stream_url = 'http://host:81/livestream.cgi?user=admin&pwd=&streamid=0'

cap = cv2.VideoCapture(stream_url)

if cap.isOpened():
    ret, frame = cap.read()
    if ret:
        # Write to local file
        cv2.imwrite("capture.png", frame)
        
        # Example: Create memory file object to save to AWS S3
        # import io
        # img_data = io.BytesIO(cv2.imencode('.png', frame)[1])

cap.release()
```
