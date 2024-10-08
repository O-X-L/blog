.. _2024-07-29:

.. include:: ../_include/head.rst

===========================================
2024-07-29 | USB Camera Steaming via WebRTC
===========================================

Project
*******

OXL Screen-Cloud Agent

----

Intro
*****

A customer expressed the wish to connect two locations with a 'screen-portal'.

It should function like the `'Dublin-NewYork' Portal <https://www.ireland.ie/en/usa/the-portal-connecting-dublin-and-new-york-city-in-real-time/>`_.

----

The path
********

There are several well-known streaming protocols that can be used for this purpose.

Some of them:

* `RTSP <https://en.wikipedia.org/wiki/Real-Time_Streaming_Protocol>`_ (*known for use with IPCams*)

* `WebRTC <https://webrtc.org/>`_ (*known for use in Web-Apps*)

* `RTMP <https://en.wikipedia.org/wiki/Real-Time_Messaging_Protocol>`_

* `HLS <https://www.cloudflare.com/de-de/learning/video/what-is-http-live-streaming/>`_

* `SRT <https://en.wikipedia.org/wiki/Secure_Reliable_Transport>`_

Prerequisites
=============

* The Agent is programmed in `GoLang <https://go.dev>`_ - therefore a solution in this programming language would be easy to integrate.

* The agent uses a Chromium browser to the display its content. The portal should also be streamed via this browser.

* The solution should, for now, only support direct connections over private networks.

WebRTC
======

Nowadays, WebRTC is often used by meeting-software.

`The Protocol <https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols>`_ allows clients to communicate directly via a peer-to-peer connections.

As a large amount of data is transferred during streaming, this is very useful for keeping the load on expensive server- and network-infrastructure low.

For this to work, however, the clients must establish their communication via an **ICE** server.

If the clients are not on the same internal network, a **STUN** server must also be used to solve the problem with P2P connections via NAT.

You can use public ICE/STUN servers. However, these could cause problems in terms of privacy.

It is possible to provide this infrastructure ourselves, but we want to save ourselves the effort of maintaining an HA setup if possible.

Pion WebRTC
===========

At first we found the `Pion organisation on GitHub <https://github.com/pion>`_.

They focus on Streaming/Real-Time protocols/implementations.

See:

* `WebRTC Examples <https://github.com/pion/webrtc/tree/master/examples>`_
* `WebRTC Mediadevices <https://github.com/pion/mediadevices/tree/master/examples/webrtc>`_

However, after a few hours we were still unable to implement their solutions without dedicated ICE/STUN servers.

Blueenviron
===========

After that we found the `Blueenviron organisation on GitHub <https://github.com/bluenviron>`_.

The have a 'Media Proxy' in their repoitoire: `MediaMTX <https://github.com/bluenviron/mediamtx>`_

As we already had some experience with proxy solutions, we could immediately imagine that we could realise our special requirements with it.

In the end, we were able to simply share USB cameras via a small HTTP server.

1. Test the Video Input:

.. code-block:: bash

    ffmpeg -f video4linux2 -list_formats all -i /dev/video0
    ffmpeg -f v4l2 -i /dev/video0 -input_format yuyv422 -video_size 640x480 -c:v copy /tmp/test_1.mkv


2. Modify the `Config-File <https://github.com/bluenviron/mediamtx/blob/main/mediamtx.yml>`_:

.. code-block:: yaml

    ...
    rtsp: yes
    ...
    rtmp: no
    ...
    hls: no
    ...
    webrtc: yes
    webrtcAddress: :8889
    # if you want to enable https
    webrtcEncryption: no
    webrtcServerKey: server.key
    webrtcServerCert: server.crt
    ...
    srt: no
    ...

    paths:
      cam:  # NOTE: the exact settings might need some tweaks for optimal usage/quality/performance
        runOnInit: >
          ffmpeg -f v4l2 -i /dev/video0 -c:v libx264 -pix_fmt yuv422p
            -preset ultrafast -b:v 600k -max_muxing_queue_size 1024
            -f rtsp rtsp://localhost:$RTSP_PORT/cam
        runOnInitRestart: yes

      all_others:


Now you can start `the Proxy <https://github.com/bluenviron/mediamtx/releases>`_ and live-stream the camera over your Browser: :code:`http://<HOST-IP>:8889/cam/`

.. include:: ../_include/user_rath.rst
