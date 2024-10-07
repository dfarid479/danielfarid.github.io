---
layout: post
title:  "Exploring Meshtastic"
subtitle: "Radio Communication and the Digital Forensics Impact"
date:   2024-10-02 19:43:59 -0400
categories: Exploring Meshtastic
background: '/img/posts/mesh.jpg'
---

 <p>I recently got started on an exciting new project: building my Meshtastic network! Inspired by a fantastic video by DataSlayer on YouTube (https://www.youtube.com/watch?v=40llxjrIG3w), I decided to dive into this innovative platform using the same hardware showcased in the video. </p>

<h2 class="section-heading"> Getting Started </h2>

<p> For my build, I chose the ESP32 board, which is known for its versatility and performance in various applications, including radio communication. While I encountered a few hiccups while pushing the firmware onto the board, I was able to resolve these issues quickly and get back on track. I chose a board that does not support GPS. I am gaining a lot more exposure to the PenTesting world and I saw the GPS function as a possible vulnerability to anonymity in the future; I figured with downloaded maps on your device you could send your location in the body of the message if necessary. I can always upgrade later and use these ESP32s for another development project. </p>

<h2 class="section-heading"> Why Meshtastic? </h2>

<p> I was drawn to this project not just for the fun of it, but also to enhance my knowledge of radio communication systems. The ability to create a mesh network that enables devices to communicate without traditional internet infrastructure is fascinating. This skill set will be invaluable as I explore the intersection of technology, digital forensics, and penetration testing in the future.

<p> Meshtastic is particularly intriguing because it offers a decentralized communication system, which could serve as a robust alternative in emergency situations or areas with poor connectivity. Plus, the community around Meshtastic is vibrant and supportive, making it easier to troubleshoot and share ideas. </p>

<p>In an emergency, communication is key especially as we become ever more reliant on our cellphones. Think hiking? Going to a large sporting event? Live in a remote location? The recent CROWDSTRIKE outage illustrated how interconnected electronic service providers are on the backend, and this event wasn’t even an attack. With your own metastatic network, if you have power, you have comms. </p>

<h2 class="section-heading"> The Build Process </h2>

<p> Following the tutorial from the video, I gathered all the necessary components and set up my workspace. It was a satisfying experience assembling everything, from wiring the hardware to flashing the firmware. My initial challenges with the ESP32 were mostly related to software compatibility. The sky's the limit with customization but I went pretty basic to begin. </p>

<p> Once I got the firmware running, the real fun began! Testing the range and reliability of the network proved to be both thrilling and informative. I was able to send messages between devices seamlessly, reinforcing the functionality of the mesh network. </p>

<h2 class="section-heading"> Testing Process </h2>

<p> My theory going into this process was that content-based artifacts would be extremely limited on Full Filesystem extractions and virtually nonexistent on Logical extractions. Your cellphone communicates via Bluetooth with the mesh network node which transmits the data via LoRa. I assumed that if available the SQL database on the handheld device would have no reason to store historically irrelevant content. Think “walkie-textie”. </p>

<p> I utilized an Apple iPhone 14 Pro, an Apple iPhone XR, and a Samsung SM-S134DL. I chose these devices better to understand the application's workings across Android and iOS. I performed the testing over a week while traveling for training so the location data would be easily visible during analysis.</p>

<p>A morning run tuned out to be a good opportunity to perform a brief range test. With a slightly obstructed line of sight in a suburban setting I was able to send and receive messages a little under 2 km away, albeit reception was spotty and several messages started getting dropped around the 1.2 km mark. I think this could be drastically improved with an upgraded antenna and/or elevation and a clear line of sight. It should be noted, that this was performed utilizing the application’s built-in range test and Whip antennas on both nodes.</p>

<p> Every few days I utilized each of the devices to send and receive messages utilizing 2 nodes in close proximity to each other just for ease of testing. I also used the messages to push the location of the handheld device.</p>

<p> After returning to the lab, I was looking forward to extracting the iPhone XR and Samsung SM-S134DL. Unfortunately the Samsung was not supported for a Full Filesystem so I performed an Advanced Logical and Android ADB Backup extractions. A Full Filesystem extraction with the keychain was acquired for the iPhone XR. The Samsung SM-S134DL extractions showed no data of Meshtastic currently or ever installed on the device. This is because the nature of the Advanced Logical and ADB Backup extractions do not acquire the necessary SQL databases.</p>

<p> The results of the iPhone XR were much more interesting. I utilized Cellebrite and Axiom to view the data from the device. I performed a general search for “Meshtastic” and “mesh” and filtered the results. It is important to note that the processing tools used did not parse the information from Meshtastic in communications, messages, or chats which means it could be easily overlooked. For testing purposes I enabled push notifications, due to this I was able to view the contents of the messages in the /userNotificationEvents directory. If I had not had push notifications, this content would not be easily visible. I located the two nodes as connected Bluetooth devices. Investigators should pay special mind to the name of the node, as this name will show up in database files and can be altered by the owner. I was able to locate a log file(\private\var\mobile\Containers\Data\Application\97276139-FED3-47CC-840B-CEAAE6C07AA9\Documents\mesh.log) for each connected node which contains configuration, location, connectivity, and transmission data. Rebuilding databases is not one of my strengths but I was able to locate (\private\var\mobile\Containers\Data\Application\-97276139-FED3-47CC-840B-CEAAE6C07AA9\Library\Application Support\Meshtastic.sqlite) an application SQL table which had a wealth of information including Channel lists (zchannelentity), Node Position (zpositionentity), and Message Content (zmessageentity). I was only able to locate my final message before terminating the test, with a better understanding of databases further message content is possible but unlikely. </p>

<p> Future investigators should take special note of locations and timestamps if a device has or is utilizing an application like Meshatastic. On multiple occurrences, Cellebrite/Axiom misread the application data as UNIX time and associated false time stamps. Also, because the location information contained in the application may be related to the Bluetooth-connected node, a node on the network, or Bluetooth-connected mobile device the presence of a location does not place the device there, further corroboration will be needed. </p>

<p> For future tests, I would make several changes before initiating the test. In the future, I would like to test the range using real-world scenarios and not the built-in testing function. Also because these projects are self-funded, I wont be destroying my new toys but a chip-off of a Mesh node may provide vital information to marry alongside acquired mobile data. The most obvious misstep was choosing an Android device which was not supported for a Full Filesystem extraction. Unfortunately, this error stopped me from being able to compare the application's interactions on Android vs. iOS.</p>

<h2 class="section-heading"> Challenges for Digital Forensics </h2>

<p> While the Meshtastic project is an exciting venture, it also brings to light significant challenges for digital forensics professionals. The decentralized nature of mesh networks can complicate investigations in several ways:

<ul>
    <li> 1. Data Recovery: Traditional communication systems often leave digital traces, such as logs or metadata, that can be analyzed. In contrast, mesh networks can operate without centralized servers, making it harder to trace messages back to their source or retrieve historical data. </li>
    <li> 2. Anonymity and Encryption: Many mesh networking applications prioritize privacy and encryption. While this is great for user security, it can hinder forensic investigations that rely on accessing clear, unencrypted data to understand communication patterns and gather evidence. Simply locating and identifying a node on the network poses challenges.</li>
    <li> 3. Device Mobility: The ability of devices to move freely within a mesh network adds another layer of complexity. If a device is part of a mesh network and then moves out of range, reconstructing communication logs or timelines can become problematic, especially if the devices were not consistently logging data.</li>
    <li> 4. Interference with Conventional Forensics: Digital forensics often relies on conventional network monitoring tools. Mesh networks, by design, operate outside traditional infrastructures, which can limit the effectiveness of these tools. </li>
</ul>

<h2 class="section-heading"> Looking Ahead </h2>

<p> As I continue to experiment with Meshtastic, I aim to explore more advanced features and applications. I’m particularly interested in how this technology can be integrated into digital forensics as these applications become increasingly more cost efficient and user friendly.</p>

<p> The possibilities are endless, and I’m excited to share my progress and insights as I delve deeper into the world of radio communication and Meshtastic.</p>

<p> If you’re interested in exploring this technology or want to learn more about my journey, feel free to reach out. </p>