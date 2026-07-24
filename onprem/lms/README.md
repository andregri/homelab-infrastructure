# Run squeezelite locally
Download the server:
```
wget https://downloads.lms-community.org/LyrionMusicServer_v9.1.0/lyrionmusicserver_9.1.0_amd64.deb
```

Install lyrion server:
```
su -
dpkg -i lyrionmusicserver_9.1.0_amd64.deb
```

on root user:
```
alsactl init
```

on standard user:
```
pulseaudio --start
squeezelite -s 127.0.0.1
```

Play a audio file
```
mplayer -volume 80 /opt/music/sample2.mp3
```