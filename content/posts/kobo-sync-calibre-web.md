+++
date = '2025-01-01T08:37:18-05:00'
draft = false
title = 'Setting up Kobo sync with Calibre Web'
+++

There I was, minding my own business downloading ebooks from my Calibre server to my Kobo Libra 2 via the web browser when I stumbled upon a comment. This comment, courtesy of a now unknown Redditor, briefly mentioned something I'd never heard of. Apparently, Calibre Web, the very service I used to sideload books onto my Kobo, had a built-in Kobo sync feature.

I stared incredulously at this comment, and then started to do some digging. I even ventured onto Google to find answers, as DuckDuckGo proved unable to pinpoint the exact information I needed. Sure enough, I found what I wanted, or at least the beginnings of it.

However, the road would prove to be a long one filled with trial and error, as these paths so often are. Anyone who has worked with anything related to Calibre knows how finnicky things can be, especially with Docker containers, reverse proxies, and Cloudflare tunnels involved. In the end, I have a 90% functional sync system set up between my Calibre library and my Libra 2.

I'll get into the two hangups a bit later, but first I want to detail what I set out to do and how I did it. As usual, complete documentation will follow later on.

## What does this setup involve?

Simply put, the key parts of this setup are: a pre-existing Calibre library, a Calibre Web installation, and a Kobo e-reader. As I've already mentioned, I did this on a Libra 2, but it should be doable on any modern Kobo e-reader. Your mileage, however, may vary.

In my homelab, I have [Calibre](https://hub.docker.com/r/linuxserver/calibre) and [Calibre Web](https://hub.docker.com/r/linuxserver/calibre-web/) running in two Docker containers on a Debian virtual machine. (I use Proxmox as my hypervisor.) All of my ebooks live on my NAS with a folder linked to the Calibre container for automatic synchronization. Any book I place in this folder gets automatically added to my library—though Calibre itself struggles with this sometimes, often requiring me to restart the software via the CTRL+R shortcut.

Accomplishing this setup was a whole headache on its own and is therefore outside the scope of this post. Suffice to say, I'm quite happy with how it works, but I will save that write-up for a different day.

Calibre Web links to my existing Calibre database and serves up my library in a more user-friendly format. It works well on mobile, desktop, and, yes, on my Kobo. It allows for other organization, ratings, and status marks. It's really the frontend that Calibre has always needed.

## Get Calibre Web ready to sync

Once you have a working Calibre Web installation going, you need to head to the Admin settings to turn on the Kobo sync feature. Under the **Configuration** section, click the button that says *Edit Basic Configuration*. Once there, expand the menu for *Feature Configuration*. Tick the box for *Enable Kobo sync*, then also select the box for *Proxy unknown requests to Kobo Store*. Make sure that the *Server External Port* in the following box matches Calibre Web's port. (The default should be `8083`.)

Save, then head to your user's settings menu. Two new options should have appeared, **Kobo Sync Token** and *Sync only books in selected shelves with Kobo*. I highly recommend checking the latter so that you don't sync every single book in your library to your e-reader, unless that is your intention.

Finally, before moving over to the Kobo itself, click the *Create/View* button under the **Kobo Sync Token** section. This will create an API endpoint URL string that you'll need shortly. Copy it.

Note: If you use an external URL for your Calibre Web installation, you might run into trouble with the Kobo syncing properly. Many users have reported problems when using a reverse proxy and I have not gotten it to work with a Cloudflare tunnel. So you might have to sub in the Calibre Web server's local IP address with the port appended before the API key.

## Configuring the Kobo e-reader

Before leaving Calibre Web, remember that checkbox for *Sync only books in selected shelves with Kobo*? Assuming you left it on, you need to create and setup shelves for syncing. How you do this is up to you as we all have our own methods for organizing our media. I went with a few shelves: Fantasy, Sci-Fi, Computer Science, Cooking, and Witchcraft. The books in the latter are my wife's and I do not want them on my Kobo, which is why I checked the box in the settings. 

If you haven't created any shelves yet, click the big **Create a Shelf** button on the left panel. In the page that follows, name the shelf whatever you want (it will appear as named on your Kobo, so name wisely) and check the box for *Sync this shelf with Kobo device*. If, however, you already have some shelves, simply go to the shelf itself, then click the **Edit Shelf Properties** button. You'll get a page that looks like the new shelf one where you can change the name and, most importantly, tell Calibre Web to sync it to your Kobo.

Now, plug in your Kobo to your computer. Follow the onscreen prompt to enable the connection then open your Kobo's root folder on your computer. You'll need to have hidden files enabled (unless you're doing this via the command line, which is an option), because you have to head to the `.kobo` folder. Click into the `Kobo` subdirectory, then look for the `Kobo eReader.conf` file. I highly suggest making a copy in that directory in case you want to revert to stock settings without fuss.

Under the `OneStoreServices` section in the configuration file, you're looking for the line that starts with `api_endpoint=`. This is where you will paste the API URL you generated earlier. Save the file, exit your text editor, and safely eject the Kobo.

Once you're back on the Kobo home screen, trigger a manual sync by tapping the Sync icon in the top toolbar. This first one might take a while as Calibre Web builds its own database and syncs everything you selected. The process should download the books themselves, as well as the metadata and covers.

## Final thoughts

So far, this system has far exceeded the old way I did things, which was to visit Calibre Web in my Libra 2's web browser and download each book manually. If I wanted to deal with connecting to my PC, I could sideload multiple items that way, too. Now? I don't have to worry about it. I did, however, run into random errors with my external URL and I believe it has to do with how I access my self-hosted services externally.

However, there is a limitation to note, more of a nuisance than anything else. Heading to the Kobo Store, you might notice that most of the book covers are missing, replaced with Kobo's generic white page with the title. This will very likely annoy many — it certainly irritates me.

One way to get around this is to buy the book you want on Kobo's website (or in the Android app), then download the book, and strip it of its DRM while importing it into Calibre. Then it'll sync to your Kobo e-reader. With this, you still have the book in your Kobo library should you decide to revert all of the changes you made here. Now you see why I suggested to make a copy of the original config file.

This same problem applies to the Overdrive section, too. Even after mapping an external port to the container's port `80`, telling Calibre Web to proxy unknown requests via port `80`, and adding the host port to the image paths — `image_host`, `image_url_quality_template`, and `image_url_template` — in the config file, I still can't get all store/Overdrive covers to download. Some do, but not all of them, even after an account repair and forced sync on the Calibre Web side. (I even exposed port `80` on the host since the Kobo config file appends the server's external port to the URL.)

I initially ran into issues using my external Calibre Web URL in the config file, but that was because I had used HTTPS, thinking this the right path given that I tunnel my Calibre Web through a Cloudflare tunnel. So chalk this one up to user error. 

Do I suggest this after hours of tinkering and debugging? Absolutely. Kobo is already pretty friendly to users, but I still prefer owning all of my books in Calibre. Whether you acquire your whole library legitimately or not is up to your conscience, but Calibre nonetheless offers a lot to get excited about. And with the Kobo sync feature in Calibre Web, I'm extremely pleased with things so far. The lack of images in the store/Overdrive irritate me, yes, but it's such a minor thing.
