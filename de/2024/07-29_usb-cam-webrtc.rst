.. _2024-07-29:

.. include:: ../_include/head.rst

===========================================
2024-07-29 | USB Camera Steaming via WebRTC
===========================================

Projekt
*******

OXL Screen-Cloud Agent

----

Intro
*****

Ein Kunde hat den Wunsch geäußert, er wolle zwei Standorte mit einem 'Portal' verbinden.

Dieses soll ähnlich wie das `'Dublin-NewYork' Portal <https://www.ireland.ie/en/usa/the-portal-connecting-dublin-and-new-york-city-in-real-time/>`_ funktionieren.

----

Der Weg
*******

Es gibt einige bekannte Streaming Protokolle, die für einen solchen Einsatzzweck genutzt werden können.

Einige davon sind:

* `RTSP <https://en.wikipedia.org/wiki/Real-Time_Streaming_Protocol>`_ (*bekannt für den Einsatz bei IPCams*)

* `WebRTC <https://webrtc.org/>`_ (*bekannt für Einsatz in Web-Apps*)

* `RTMP <https://en.wikipedia.org/wiki/Real-Time_Messaging_Protocol>`_

* `HLS <https://www.cloudflare.com/de-de/learning/video/what-is-http-live-streaming/>`_

* `SRT <https://en.wikipedia.org/wiki/Secure_Reliable_Transport>`_

Voraussetzungen
===============

* Der Agent ist in `GoLang <https://go.dev>`_ programmiert - eine Lösung in dieser Programmiersprache wäre gut zu integrieren

* Der Agent nutzt eine Chromium Browser für die Anzeige. Das Portal sollte auch über diesen gestreamt werden

* Die Lösung soll, bis auf weiteres, nur direkte Verbindungen über private Netzwerke unterstützen.

WebRTC
======

WebRTC wird heutzutage z.B. von Meeting-Lösungen eingesetzt.

`Das Protokoll <https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols>`_ erlaubt Clients direkt via Peer-to-Peer Verbindung zu kommunizieren.

Da beim Streaming eine große Menge an Daten übertragen werden, ist dies sehr nützlich um die Last auf teure Server- & Netzwerk-Infrastruktur niedrig zu halten.

Damit dies funktioniert, müssen die Clients jedoch über einen **ICE** Server deren Kommunikation herstellen.

Falls die Clients nicht im selben internen Netzwerk sind, muss auch ein **STUN** Server genutzt werden, um das Problem mit P2P-Verbindungen über NAT zu lösen.

Man kann öffentliche ICE/STUN Server nutzen. Doch diese könnten Probleme im Bezug auf Privatsphäre verursachen.

Diese Infrastruktur selbst bereitzustellen ist möglich, doch der Aufwand der Wartung eines HA-Setups wollen wir uns, wenn möglich, einsparen.

Pion WebRTC
===========

Vorerst sind wir noch einiger Suche auf die `Pion Organisation auf GitHub <https://github.com/pion>`_ gestoßen.

Sie fokusieren sich auf Streaming/Real-Time Protokolle.

Siehe:

* `WebRTC Examples <https://github.com/pion/webrtc/tree/master/examples>`_
* `WebRTC Mediadevices <https://github.com/pion/mediadevices/tree/master/examples/webrtc>`_

Doch nach einigen Stunden konnten wir deren Lösungen noch nicht ohne dedizierte ICE/STUN Server implementieren.

Blueenviron
===========

Danach sind wir auf die `Blueenviron Organisation auf GitHub <https://github.com/bluenviron>`_ gestoßen.

Sie haben einen 'Media Proxy' in ihrem Repertoire: `MediaMTX <https://github.com/bluenviron/mediamtx>`_

Da wir schon einige Erfahrungen mit Proxy-Lösungen gemacht haben, konnte wir uns gleich vorstellen, dass wir unsere speziellen Anforderungen damit umsetzen könnte.

Schlußendlich konnten wir USB-Kameras einfach über einen kleinen HTTP Server teilen.

1. Video Input Testen:

.. code-block:: bash

    ffmpeg -f video4linux2 -list_formats all -i /dev/video0
    ffmpeg -f v4l2 -i /dev/video0 -input_format yuyv422 -video_size 640x480 -c:v copy /tmp/test_1.mkv


2. Das `Config-File <https://github.com/bluenviron/mediamtx/blob/main/mediamtx.yml>`_ anpassen:

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


Nun kann man `den Proxy <https://github.com/bluenviron/mediamtx/releases>`_ starten und die Kamera über den Browser Live-Streamen: :code:`http://<HOST-IP>:8889/cam/`

.. include:: ../_include/user_rath.rst
