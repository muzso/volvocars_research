# Request for help in security research

I am looking for owners of recent Volvo and Polestar models, seeking your help to better characterize a security vulnerability that I found while digging into the software stack of my own vehicle, a Volvo EX30. Before I make the responsible disclosure to Volvo Cars, I'd like to confirm whether it affects other models as well.

This is where I need some help: I need access to the software running on the AAOS (Android Automotive OS) infotainment systems of potentially affected models. Of course only the system part of the head unit, nothing that is individual to a vehicle (device/vehicle ID, etc.) or user generated data.

The particular brands/models I'm interested in:

- all Volvo Cars models that use AAOS in their infotainment system (vehicles made in the last 3-5 years or even older models that might have gotten it as an upgrade)
- all Polestar models that use AAOS in their infotainment system (vehicles made in the last 5-6 years)

I suspect that the issue predates the arrival of AAOS, but I'm not really familiar with the Sensus architecture. Still, if somebody has a file system dump from an older Sensus infotainment system, I'd like to take a look at it too.

Here's what I need:

- a dump of parts of the file system like `/product`, `/system`, `/system_ext`, `/vendor`, etc. (or an OTA update package, i.e. `*.VBF` files)
- name of the vehicle model
- model year
- software version

If you already have a full or partial dump of an AAOS file system (from a Volvo/Polestar) and you'd be willing to share it with me, please do so.

If you don't, but you do have a Volvo car with AAOS onboard and are willing to help (or you're just interested in similar things yourself), I wrote an app that can extract as much data from a head unit (aka. infotainment) as any app available through the Play Store could. It doesn't request (or use) any storage-related Android permissions, so it can read only files that the OS (Android) and/or the vendor didn't consider sensitive enough to protect from third-party apps. This means that the app should not be able to access user data by design, and even if it could, the app uses a path exclusion list to avoid looking for user data in the first place.

I've shared the source code of the app here: [https://github.com/muzso/android_system_dumper](https://github.com/muzso/android_system_dumper)

You can inspect, download and compile it yourself, and install it on any AAOS vehicle (using a Google developer account and Google Play's Internal Testing track) or I can give access to the app through my own Internal Testing track. This way, anybody without software development experience can use it.

I've put up a couple of videos to demonstrate how it works.

It supports two scenarios for getting the files off the device:

1. Upload encrypted ZIP bundles to a file sharing service: [https://www.youtube.com/watch?v=878IzMO6CiQ](https://www.youtube.com/watch?v=878IzMO6CiQ)
2. Download ZIP bundles over a direct Wi-Fi connection between two devices: [https://www.youtube.com/watch?v=4zX-aR7sUuw](https://www.youtube.com/watch?v=4zX-aR7sUuw)

The latter is a lot faster, but requires a bit more technical experience (like setting up a Wi-Fi hotspot on your phone and connecting the vehicle to it).

By default the app uses:

- the Tor network to help preserve the anonymity for the vehicle owner (it is highly unlikely that anybody could tell where the upload came from)
- standard ZIP files encrypted with a long random passphrase (this is for compatibility with the built-in ZIP support of Windows, but AES-encrypted ZIPs are supported as well)
- double-zipping prevents someone without the passphrase from seeing the file listing

You can reach me using any one of the following methods:

- on Signal via the "muzso.01" username (for the privacy conscious)
- email: zsmuller at proton dot me
- or as a last resort: here via DM

If I don't get any help with this during the next month or so, I'll just report to Volvo Cars what I've found regarding the EX30. Later on other like-minded Volvo owners will either confirm or deny whether other models are affected or not.

Thanks in advance to anybody who's willing to pitch in with this effort.

P.S. Here's a little background on me ...

I've been a core contributor to the Volvo EX30 sub for the last 2 years. I've been a software engineer and a cybersecurity enthusiast for over 25 years, working with cybersecurity professionally for the last 5 years. You can look me up: my GitHub account has my name and from there it's easy enough to find my Linkedin profile, etc.

I bought an EX30 approx. two years ago and right from the start I had a hunch that one of the vehicle's features might contain a security vulnerability (as in cybersecurity, not to be confused with safety). I didn't give much thought to it for a while, until a little over a year ago I found proof while investigating something different. I've been sitting on this for a while now (I hoped that it would be fixed by the company on its own), but it's time to share this with Volvo Cars and give them the opportunity to mitigate the issue before publishing my findings. My primary goal is to get this fixed within a reasonable timeframe (90 days is a common industry standard in cybersecurity), but if they decline to fix it, I won't wait indefinitely. The vulnerability will still be there for anyone with sufficient knowledge to discover. IMHO not taking it public would be more irresponsible.

I won't share any details until I've talked it over with Volvo Cars and agreed with them on a timeline, so please don't even ask. Even a small detail could help a malicious actor focus their attention and efforts in the right direction, which is something I definitely want to avoid.

I don't know of any mitigation that owners could apply themselves, which makes this issue all the more severe.

Since the feature is shared by other Volvo Cars models, I'd like to see if any other models are vulnerable too. Having factual proof either way would be useful if the company disagrees with my assessment. Hence this request for help.
