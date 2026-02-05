---

title: Read H264 Livestream

category: Gist
tags: [ Geek]
date: 2019-02-28
---

Got a Solar WiFi IP camera from <a href="https://www.ebay.com.au/itm/302918525683?ul_noapp=true">Wireless Solar IP67 Security Camera System Outdoor Home Cam 1080P 2MP</a>

I wanted to capture video or images from the camera using a script or program. 


`ffmpeg -re -i "http://host ip:81/livestream.cgi?user=admin&pwd=&streamid=0" -c copy -f mpegts test.mp4 `

`ffmpeg  -i "http://host ip:81/livestream.cgi?user=admin&pwd=a123&streamid=0" -c copy -f mpegts -ss 5 -frames:v 5 testt.png`

FFMPEG is not portable. Another way is to use cv2 to capture a picture:
(ref <a href="https://stackoverflow.com/questions/28477600/not-able-to-play-h264-video-on-opencv">Not able to play .h264 video on OpenCV?</a>)

```python
cap = cv2.VideoCapture('http://host:81/livestream.cgi?user=admin&pwd=&streamid=0')
        while(cap.isOpened()):
            ret, frame = cap.read()
            # write to local file
            cv2.imwrite("capture.png",frame)
            # create memory file object save to AWS S3
            #img_data = io.BytesIO(cv2.imencode('.png', frame)[1])
            break
        cap.release()
``` 

