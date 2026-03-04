
# 🎵 The Ultimate Personal Music Server
A Self-Hosted Music Streaming Ecosystem Built On A Raspberry Pi. This Setup Features Automatic High-Res Metadata Fetching, Global Remote Access Via ZeroTier, And A Bulletproof Mobile Offline Experience.

## ⚖️ Legal Disclaimer
This project is for **personal use only**. Users are responsible for ensuring they own the rights to the music files they upload. Kyle8973 does not condone or support music piracy. Use this setup to stream your own purchased or royalty-free collection.
-   The author does not condone or support the unauthorised downloading or distribution of copyrighted material.
    
-   **Format Shifting:** In some jurisdictions (like the UK), digitising physical CDs you own currently exists in a legal grey area or is technically prohibited.
    
-   **Responsibility:** By using this guide, you agree that you are solely responsible for complying with the copyright laws of your specific country.

---

## 🛠️ Server Setup
### 1. Project Structure
```bash
mkdir ~/music-server && cd ~/music-server  # Create and enter the project root
mkdir data music                           # Create folders for the database and your songs
```

### 2. Docker Compose
If you do not already have Docker Compose installed on your server, you can do so by following the commands below:
-   **Install Docker Engine:**
    ```bash
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh
    
    ```
    
-   **Install Docker Compose Plugin:**
    ```bach
    sudo apt-get update
    sudo apt-get install -y docker-compose-plugin
    
    ```
    
-   **Permissions:** Add your user to the Docker group so you don't need `sudo` for every command:
    ```bach
    sudo usermod -aG docker $USER
    
    ```  
 Note: Log out and log back in, or run `newgrp docker` to apply this change.
    <br>
    
Create your docker-compose.yml inside the `music-server` folder.

This can be done using the following command and pasting the code below:
```bash
 sudo nano music-server/docker-compose.yml
 ```

Note: Replace the [Last.fm](https://www.last.fm/api/account/create) and [Spotify](https://developer.spotify.com/dashboard) keys with your own.

```yaml
# Paste In music-server/docker-compose.yml
# Kyle8973
# https://github.com/Kyle8973/Personal-Music-Server
version: "3"
services:
  navidrome:
    image: deluan/navidrome:latest         # Pulls the latest stable Navidrome image
    user: 1000:1000                        # Runs the container as your Pi's local user for permission safety
    ports:
      - "4533:4533"                        # Maps the server port (Host:Container)
    restart: unless-stopped                # Automatically starts the server if the Pi reboots
    environment:
      # Metadata Integration (Last.fm and Spotify API keys required)
      ND_SPOTIFY_ID: "your_id"             # Your Spotify Developer Client ID
      ND_SPOTIFY_SECRET: "your_secret"     # Your Spotify Developer Client Secret
      ND_LASTFM_ENABLED: "true"            # Enables Last.FM
      ND_LASTFM_APIKEY: "your_key"         # Your Last.fm API Key
      ND_LASTFM_SECRET: "your_secret"      # Your Last.fm API Secret
      
      # UI & Performance
      ND_LOGLEVEL: "info"                  # Sets logging level (info/debug/error)
      ND_COVERARTPRIORITY: "embedded, cover.*, folder.*, front.*, external" # Priority list for finding album art
      ND_IMAGECACHESIZE: "100MB"           # Reserves 100MB of RAM to make scrolling album art faster
    volumes:
      - "./data:/data"                     # Maps the local 'data' folder for the database/cache
      - "./music:/music:ro"                # Maps your 'music' folder as Read-Only for safety

  filebrowser:
    image: filebrowser/filebrowser:latest  # Pulls the FileBrowser image
    user: 1000:1000                        # Runs as local user to match file permissions
    ports:
      - "8080:80"                          # Access your files via browser at Pi-IP:8080
    volumes:
      - "./music:/srv"                     # Allows you to upload music directly to the music folder
      - "./filebrowser.db:/database.db"    # Saves your FileBrowser settings and user
  ```

Now run the following commands:
```bash
touch filebrowser.db 
chmod +x setup.sh && ./setup.sh 
docker compose up -d

# This Command Will Show Your Your Password For FileBrowser (Write It Down, We Will Need It Later)
docker compose logs filebrowser | grep -i 'password'
```
 
---
## 🔒Remote Access (ZeroTier)
Think of ZeroTier as a private VPN just for your devices. It lets your phone talk to your server as if they were on the same Wi-Fi network, without opening ports or weakening your firewall security.

**1 - Create A ZeroTier Account:**
Go to [ZeroTier Central](https://my.zerotier.com/) and make an account if you do not already have one.

**2 - Create A Network:**
Click the "Create A Network" button, you should that a network has now been created with a random name such as "adoring_taylor".

**3 - Configuring The Network:**
Click on the network you have just created, we will now configure a few settings to get our ZeroTier network up and running.

**3.1:** Firstly, click the settings dropdown, this is located under the members dropdown, within these settings you should see your network ID, network name / description and access control.

For ease of use, you can edit the network name to something more suitable such as "Music Server".

Ensure that the "Access Control" is set to private, this prevents unauthorised users from joining the network and being automatically authenticated.

**3.2:** Now we will configure our "Managed Routes", scroll down and under settings you should see "Advanced", this shows our managed routes, the IP address that will be assigned to clients within our ZeroTier network.

A managed route should have been automatically selected for you upon creation of the network, in my case I can see `10.147.19.0/24 (LAN)`. However, you can change this by selecting another option from the "Easy" section under IPv4 Auto Assign such as `192.168.193.*`, simply select what works best for you or leave it as is.

That is the end of our configuration.

**4 - Joining The Network:**
<br>
**4.1 - Server:** On your server, such as your Raspberry Pi, install ZeroTier and join the network.
```bash
curl -s [https://install.zerotier.com](https://install.zerotier.com) | sudo bash
sudo zerotier-cli join [Your_16_Digit_Network_ID]
```
Note: Replace `[Your_16_Digit_Network_ID]` with the Network ID shown in ZeroTier

**4.2 - Mobile Device:** On your mobile device install the ZeroTier app ([Android](https://play.google.com/store/apps/details?id=com.zerotier.one&hl=en) / [iOS](https://apps.apple.com/us/app/zerotier-one/id1084101492)) and click "Add Network", paste the same Network ID into the app or you can scan the QR code shown on the ZeroTier dashboard.

Leave all other options such as DNS configuration and IPv4 DNS blank, unless you specifically need these for your use case.

Click "Add" and your mobile device should now be added to the ZeroTier network, you should now see it on the home page.

**4.3 - Authorising Our Devices:** Now that both our server and mobile device are added, we need to authorise them within ZeroTier, this grants them the permission to connect to our network.

To do this go back to the ZeroTier dashboard and go to the "Members" dropdown, you should see 2 devices with "Auth" showing (🚫), other details such as the device MAC Address, Managed IPs and Physical IP will also be shown.

Click the icon under the "Edit" heading, you should see "Authorized" at the very top alongside a checkbox, tick the box.

You can also edit the name and description for each device to allow for easier navigation. For example, I set the name KylePi for my server (Raspbery Pi) and Kyle's Pixel 9 for my mobile device.

Click save and both devices should now show a (✅) under the "Auth" heading.

Now go back to the mobile app and you should see a switch icon displayed next to the network, click it and a notificaiton should pop-up confirming you are connected, if it says unauthorised, go back to the ZeroTier dashboard and confirm the Auth shows (✅).

---
## 🎵Populating Your Music Libary
To stay 100% legal you must ensure you only upload music in which you have the rights to do so. We recommend using **DRM-free digital storefronts**:

1.  **Source Media:** Purchase and download high-quality files from:
    
    -   **[Bandcamp](https://bandcamp.com):** Direct artist support and FLAC/MP3 downloads.
        
    -   **[Qobuz Store](https://www.qobuz.com/gb-en/shop):** Excellent for high-resolution 24-bit audio.
        
2.  **Upload:** Access FileBrowser at `http://[Your-ZeroTier-IP]:8080`. <br><br> Login using the username "admin" and the password you wrote down earlier. <br><br> Forgot it? Do not worry just run `docker compose logs filebrowser | grep -i 'password'`<br><br> You can now drag and drop your music folders directly into the window. <br><br>However, I recomend sorting your music into folders such as Artist Name / Album name (Example: I created a folder called Nickelback and within that folder created another folder named Dark Horse (Album Name) as Navidrome works best this way.
    
4.  **Initialize Navidrome:** Go to `http://[Your-ZeroTier-IP]:4533` to create your admin account and start browsing your library, you should see the music you uploaded.

**Note:** `[Your-ZeroTier-IP]` refers to the ZeroTier IP of your server not your mobile device, if you are unsure you can run `hostname -I` on your server and find the IP that matches your ZeroTier subnet (Example: 10.147.19.0/24).

**Note:** You can change the default Navidrome username and password once you are logged in..
    

----------

## 💿 Digitising CDs (Optional)

If you are in a region where personal "format-shifting" is permitted (e.g., the USA), use these tools to add your physical discs:

-   **Windows:** [Exact Audio Copy (EAC)](https://www.exactaudiocopy.de/) – The industry standard for bit-perfect rips.
    
-   **Linux:** [Whipper](https://github.com/whipper-team/whipper) – A terminal-based ripper that uses MusicBrainz for perfect metadata tagging.
---
## 📱Downloading The Mobile App
Now it is time to download the mobile app which will allow us to listen to our music on the go, for Android I recomend using **Symfonium** and for iOS I recomend using **Amperfy**. Other options are available. However, I have found these options to work best.

Note:  **Pricing Comparison (As of February 2026):**

-   **Spotify Premium Individual:** £12.99 per month (approx. **£155.88/year**).
    
-   **Symfonium:** Free trial, followed by a **£5.49 one-off lifetime license**.
    

By paying for Symfonium once, you own your experience forever. In just two weeks of use, the app pays for itself compared to a monthly Spotify subscription. You get a cleaner UI, better offline management, and full control over your high-quality files without the recurring "rent" for your music.

I do believe Amperfy for iOS is completely free. However, do not quote me on that.

---
## ⚙️Configuring The Mobile App
**Disclaimer:** Since I personally own an Android device, the instructions below are for configuring **Symfonium**  on Android, for information regarding **Amperfy** iOS, I have linked the official [GitHub](https://github.com/BLeeEZ/amperfy). I apologise in advance to any iOS users.

**1:** Open the Symfonium app on your Android device, and select "On a network server or cloud provider".

**2:** From the media providers list, select (Open) Subsonic.

**3:** Input your connection details:
Server URL - `http://[Your-ZeroTier-IP]:4533` (IP of your server).
Username - The username you created in Navidrome.
Password - The password you created in Navidrome.

**Note:** Your mobile device must be connected to ZeroTier for this to work. - It may take a moment to connect, however, once connected you should see a nice looking UI displaying any music you have uploaded so far.

**🤓 Optional Smart Caching:**
Smart caching acts as an intelligent "conveyor belt" that stays ahead of your listening habits to prevent buffering and signal loss. By utilising a **Pre-cache** buffer, the app silently downloads upcoming tracks while you listen to the current song, ensuring your next few songs are always available even if you enter a dead zone. Once a track finishes, the **Rolling Cache** logic "promotes" the file from a temporary buffer into a long-term storage vault on your device, automatically building a personalised offline library of your most-played music without requiring any manual downloads.

**How it works:**
1.  **The Lookout (Pre-cache):** While you’re listening to Song A, the app looks ahead and downloads the next 3 songs (Songs B, C, and D) into a temporary "loading dock" (the Playback Cache).
    
2.  **The Handover:** Once you finish listening to a song completely, the app realises you liked it enough to hear the whole thing.
    
3.  **The Vault (Rolling Cache):** It then automatically "promotes" that song from the temporary dock to your 5GB "Rolling Cache" vault for permanent-ish offline use.
    
4.  **Auto-Cleanup:** When that 5GB vault gets full, the app automatically deletes the song you haven't listened to in the longest time to make room for your new favorites.

**Steps to set it up:**

1. Go to **Settings > Offline, Cache and Download**.
2. Go to **General** and set a rolling cache size of your choice, I recommend around 5GB
3. Enable **Automatic Offline Mode** - This ensures when you are offline, the app only shows music that can be streamed offline.
4. Disable **Only Consider Primary Connection For Wi-Fi Status** - This ensures the app works with ZeroTier and treats it as a network connection.
5. Go back to **Settings > Offline, Cache and Download**.
6. Select **Playback**
7. Set the **Playback Cache Size** I recommend around 1GB
8. Set the **Pre-cache tracks** to 3 tracks for both Mobile and Wi-Fi, you can do more however, for best performance I recommend 3.
9. Enable **Add Playback Cached Media To Offline Rolling Cache** - This ensures playback cache is promoted to the offline cache for offline listening.
10. Done! 
---
## 🎨 Troubleshooting & Metadata

-   **Blurry Covers:** In the Navidrome Web UI, go to **Activity > Full Scan**. This forces the server to fetch high-res art from Last.fm if your local files are poor quality.
    
-   **Artist - Topic:** If your tags are messy or music belonging to the same artist or album are not grouped together, use **FileBrowser** to rename the files or a tool like **[Mp3tag](https://www.mp3tag.de/en/download.html)** on Windows to clean the "Album Artist" and Album Name fields.

- **Further Support:** Should you require further support, please open an [Issue](https://github.com/Kyle8973/Personal-Music-Server/issues) on Github and I will do my best to support you.
---
## ⚠️License
This project is published under the MIT [License](https://github.com/Kyle8973/Personal-Music-Server/blob/main/LICENSE):

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE. 
  
