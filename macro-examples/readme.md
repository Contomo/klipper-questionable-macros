
# Motion logger macro
<img width="400" height="300" alt="image" src="https://github.com/user-attachments/assets/29343f2b-32a0-48fb-8c04-e63c80f14319" />

Allows you to call and start/stop recording and save logger images, i use it for motion, but any data could be logged here.

Simply edit the settings `'img_dir': '~/printer_data/config/motion_images',` to point at your folder where you want it saved, and call `MOTION_LOGGER START=1` to start logging, and `MOTION_LOGGER STOP=1` to stop logging and generate the image.
