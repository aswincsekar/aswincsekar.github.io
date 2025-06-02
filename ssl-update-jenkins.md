**Title: The SSL Cert Shuffle: Updating Jenkins on a Bitnami VM**

Alright team, gather 'round the virtual water cooler. Ever had one of those "simple" tasks that spirals a bit? That was me this week with an SSL certificate update for our Jenkins server. It's running on a VM, and as we all know, "it's on a VM" can mean a multitude of sins... or at least, configurations.

**The Mission, Should I Choose to Accept It: New SSL Cert for Jenkins**

The request was straightforward: our Jenkins SSL cert was nearing its expiration date, and we needed to swap in the shiny new one. Standard procedure, right? Famous last words.

My first thought: "Is Jenkins serving SSL directly, or is it behind a reverse proxy?" This is always Step Zero.

**Step 1: Reconnaissance - What's Listening on 443?**

A quick `sudo netstat -tulnp | grep :443` on the VM gave me the first clue:
```
tcp6       0      0 :::443                  :::*                    LISTEN      1475/httpd
```
`httpd`. Okay, so Apache is our SSL terminator. That simplifies things for Jenkins itself – it's just blissfully serving HTTP locally, and Apache is handling the encrypted front door.

**Step 2: Apache Config Safari - The Usual Suspects**

My muscle memory kicked in: `cd /etc/httpd/conf.d/` or `cd /etc/apache2/sites-available/`.
*Crickets.*
Neither `/etc/httpd/` nor `/etc/apache2/` existed.

"Hmm," I thought. "Custom compile? Weird distro?" The `ls /etc` output looked fairly Debian-esque, but no standard Apache paths. This is where you thank `ps` and `/proc`.

**Step 3: Following the Process Breadcrumbs**

Time to find out *which* `httpd` this was and where it lived.
```bash
ps aux | grep httpd
```
And there it was, clear as day:
```
root      1475  0.0  0.1  12428  2456 ?        Ss   Feb24   7:48 /opt/bitnami/apache/bin/httpd -f /opt/bitnami/apache/conf/httpd.conf
```
**Bitnami!** Of course. The self-contained, all-in-one stack. This changes the game. No more `/etc/whatever`; we're now playing in the `/opt/bitnami/` sandbox. The command even helpfully told me the exact config file: `/opt/bitnami/apache/conf/httpd.conf`.

**Step 4: Navigating the Bitnami Labyrinth**

With the main config file identified, it was time to dive in:
```bash
sudo less /opt/bitnami/apache/conf/httpd.conf
```
I started looking for `Include` directives, particularly anything that screamed "SSL" or "virtual hosts." Bitnami likes to keep things organized (thankfully). I also poked around `/opt/bitnami/apps/jenkins/conf/` just in case the Jenkins app had its own specific vhost snippet for SSL.

A strategic `sudo grep -R "SSLCertificateFile" /opt/bitnami/apache/conf/` quickly pointed me to the relevant file, which in many Bitnami setups is something like `/opt/bitnami/apache/conf/bitnami/bitnami-ssl.conf` or within `/opt/bitnami/apache/conf/extra/httpd-ssl.conf`. Sometimes, if Jenkins is a Bitnami "app," the vhost config is in `/opt/bitnami/apps/jenkins/conf/httpd-vhosts.conf` or similar, often including SSL directives.

**Step 5: The Actual Swap - Certs, Keys, and Configs**

Once I located the `<VirtualHost *:443>` block for our Jenkins instance, the rest was more familiar territory:

1.  **Backup, Backup, Backup:**
    *   `sudo cp -a /opt/bitnami/apache/conf /opt/bitnami/apache/conf_backup_$(date +%F)_pre_ssl_update`
    *   Also backed up the *old* certificate and key files themselves, just in case of a catastrophic typo on my part.

2.  **Upload New Certs:**
    *   Got the new `fullchain.pem` (certificate + intermediate chain) and `privkey.pem` (private key).
    *   Copied them to a sensible place. I opted for `/opt/bitnami/apache/conf/certs/` to keep things tidy.
        *   `sudo mkdir -p /opt/bitnami/apache/conf/certs/`
        *   `sudo cp new_jenkins.fullchain.pem /opt/bitnami/apache/conf/certs/jenkins.ourdomain.com.pem`
        *   `sudo cp new_jenkins.privkey.pem /opt/bitnami/apache/conf/certs/jenkins.ourdomain.com.key`

3.  **Permissions Check:**
    *   `sudo chown root:root /opt/bitnami/apache/conf/certs/jenkins.ourdomain.com.*`
    *   `sudo chmod 600 /opt/bitnami/apache/conf/certs/jenkins.ourdomain.com.key` (Key needs to be tight)
    *   `sudo chmod 644 /opt/bitnami/apache/conf/certs/jenkins.ourdomain.com.pem`

4.  **Edit Apache Config:**
    *   Opened the SSL vhost file identified earlier.
    *   Updated the `SSLCertificateFile` and `SSLCertificateKeyFile` directives:
        ```apache
        SSLCertificateFile "/opt/bitnami/apache/conf/certs/jenkins.ourdomain.com.pem"
        SSLCertificateKeyFile "/opt/bitnami/apache/conf/certs/jenkins.ourdomain.com.key"
        ```
    *   My new `.pem` was a fullchain, so if there was an `SSLCertificateChainFile` directive, I commented it out.

**Step 6: The Moment of Truth - Test and Restart**

Bitnami has its own control script, which is the preferred way to manage its services.

1.  **Config Test:**
    ```bash
    sudo /opt/bitnami/apache/bin/apachectl -t
    ```
    Or, more explicitly using the full path from `ps`:
    ```bash
    sudo /opt/bitnami/apache/bin/httpd -f /opt/bitnami/apache/conf/httpd.conf -t
    ```
    `Syntax OK`. Deep breath.

2.  **Restart Apache (Bitnami Style):**
    ```bash
    sudo /opt/bitnami/ctlscript.sh restart apache
    ```

**Step 7: Verification - Did It Work?**

*   **Browser Check:** Cleared cache, hit `https://jenkins.ourdomain.com`. Clicked the padlock. New expiry date? Check. Correct issuer? Check.
*   **SSL Labs:** Ran it through Qualys SSL Labs. A+? Check. (Okay, maybe not always an A+ first try, but we aim high!)
*   **OpenSSL CLI:** `openssl s_client -connect jenkins.ourdomain.com:443 -servername jenkins.ourdomain.com`. Eyeballed the cert chain. Looked good.
*   **Logs:** Tailed `/opt/bitnami/apache/logs/error_log` just to be sure no weird SSL handshake errors were popping up.

**Lessons Learned (or Re-Learned):**

*   **Assumptions are the enemy:** Don't assume standard paths, especially on VMs where anything could have been installed.
*   **`ps` is your friend:** When in doubt, see what's actually running and how it was invoked.
*   **Bitnami has its own ecosystem:** If you see `/opt/bitnami/`, remember to use `ctlscript.sh` and expect configs to be within that structure.
*   **Backup everything:** Configs, old certs. You'll thank yourself when a typo brings things crashing down.

And that's the tale of the Jenkins SSL cert update. A little more detective work than anticipated, but we got there in the end. Now, who wants coffee? My treat.
