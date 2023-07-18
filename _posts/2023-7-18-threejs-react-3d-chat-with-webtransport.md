---
layout: single
title: "Building a Virtual Reality 3D Chat with Three.js, React and WebTransport"
excerpt: "Leveraging my experience with programming languages and cutting-edge technologies like WebTransport, I aim to create an environment where imagination is the only limit."
header:
  image: /assets/images/posts/threejs-react-3d-chat-with-webtransport/header.jpg
  teaser: /assets/images/posts/threejs-react-3d-chat-with-webtransport/header.jpg
  overlay_image: /assets/images/posts/threejs-react-3d-chat-with-webtransport/header.jpg
  overlay_filter: 0.5
  og_image: /assets/images/posts/threejs-react-3d-chat-with-webtransport/og.png
date: 2023-07-18
comments: true
---

15 years ago, my friends and I were captivated by Active Worlds, a 3D chat
platform that allowed users to effortlessly create new experiences in existing
worlds, even establishing their own.

However, our enjoyment was limited by a tight budget. After exploring
alternatives such as Blaxxun, which used web browsers and VRML technologies
to connect people, I decided to create my own solution, intending to offer
more freedom and possibilities to unleash our imagination.

At the time, my programming knowledge was restricted to mIRC scripting, TCL,
and Visual Basic 6. Languages like C, C++, and Java seemed quite complex, Unity
wasn't around, and Unreal Engine was still popularly known as
Unreal Tournament 2004. Consequently, I had to put this idea on hold.

Fast forward 15 years, and I've mastered the C programming language, primarily
by delving deep into the Quake 3 source code. With the evolution of technology,
creating 3D games has become simpler than ever, and web browsers no longer need
Flash, ActiveX, or NPAPI plug-ins to run incredible software effortlessly,
without any cumbersome installations.

A month ago, Mozilla released Firefox 114, introducing the WebTransport feature.
This innovation was the missing piece I needed to enable players to transmit
their movements smoothly within 3D environments.

Employing HTTP3 and UDP, this technology is clearly the successor to WebSockets,
which relied solely on TCP. With this, WebRTC will no longer be necessary for
unreliable and fast data transmission.

Now, with all the required tools in hand, I can finally set this idea in motion
and see where it leads.

For this project, I plan to use Three.js and React for the front-end and Rust
for the back-end, as wtransport was the only library with a working
implementation of a WebTransport server at the time of writing this post.

I'll be sharing more details as progress unfolds. I look forward to reading any
feedback you might have throughout this exhilarating journey.