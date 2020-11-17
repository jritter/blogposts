# SSH Dynamic Port Forwarding

Many enterprises are using Secure Shell (SSH) jump servers in order to access business critical systems. Administrators first have to connect to a jump server using SSH, possibily through a VPN, before being able to connect to the target system. This usually works great as long as an administrator sticks with command line based administration. It gets a bit more tricky when an administrator would like to break out of the command line realm, and for instance use a web based interface.

Let us look at the following scenario: Bob is a system administrator at Securecorp, and he just got an alert, indicating that a database cluster consisting of `sirius.securecorp.io` and `orion.securecorp.io` is performing poorly. For a first analysis, he usually uses the RHEL8 web console. From his workstation, the firewall doesn't allow him to connect directly to this system, but he has the possibility to go through a jump server called `bastion.securecorp.io`.

SSH access to the database cluster is straight forward:

```bash
[bob@workstation ~]$ ssh bastion.securecorp.io
[bob@bastion ~]$ ssh sirius.securecorp.io
[bob@sirius ~]$
```

```bash
[bob@workstation ~]$ ssh bastion.securecorp.io
[bob@bastion ~]$ ssh orion.securecorp.io
[bob@orion ~]$
```

But what if Bob wants to access the RHEL8 web console of sirius and orion? There are multiple ways to achieve this goal.

**Disclaimer**: In some organizations, security policies do not allow port forwarding. To make sure that you don't breach any rules, please consult with your IT security representative.

## SSH Port Forwarding

Using SSH, Bob opens a TCP tunnel for both systems, pointing to the web console port (9090) of sirius and orion at the other side.

```bash
[bob@workstation ~]$ ssh -L 9090:sirius.securecorp.io:9090 bastion.securecorp.io
[bob@bastion ~]$
```

```bash
[bob@workstation ~]$ ssh -L 9091:orion.securecorp.io:9090 bastion.securecorp.io
[bob@bastion ~]$
```

Bob can now point the browser of his local workstation to <https://localhost:9090> to access the web console of sirius, and <https://localhost:9091> to access the web console of orion.

This approach might work well for certain cases, but has its limitations:

* **TLS Certificate validation:** The local browser is unhappy because in most cases, the certificate common name doesn't match with the hostname in the address bar (localhost), so the certificate validation fails
* **Redirects** When the website you are accessing redirects you to another URL, the connection fails because port forwarding is only valid for exactly this webserver. This might be a problem when using Single Sign On.

## Start a Browser on the Jump Server

Another approach for Bob is to start a browser such as Firefox on the jump server, and display it locally on his workstation. For that, SSH provides a feature called X forwarding, which can be used in this situation.

```bash
[bob@workstation ~]$ ssh -X bastion.securecorp.io
[bob@bastion ~]$ firefox https://sirius.securecorp.io:9090 &
[bob@bastion ~]$ firefox https://orion.securecorp.io:9090 &
```

Using this method, the browser process runs on the jump server, and the connection to the web console of sirius and orion are allowed. Only the rendering happens on the workstation of Bob.

While this approach solves some problems of plain SSH port forwarding, it also has limitations:

* **Performance:** This method usually performs rather poorly, because the graphical output has to be transfered from the jump server to the workstation, which is very inefficient.
* **Prerequisites:** A browser such as Firefox needs to be installed on the jump server, and an X server needs to be running on the workstation

## Enter Dynamic Port Forwarding

Having explored the previous two approaches and learned about the disadvantages of them, it would be great to have a third option which brings us the best of both worlds:

* Browser of the workstation can be used
* Connectivity and DNS situation should be the same as on the Jump server

To achieve that, SSH provides a feature called **dynamic port forwarding**, which leverages the [SOCKS](https://en.wikipedia.org/wiki/SOCKS) protocol. In this configuration, SSH acts as a SOCKS proxy, relaying all relevant traffic through the SSH connection. For this to happen, the client (in our example it is the Browser) needs to be SOCKS aware.

Bob can initiate an SSH Session with dynamic port forwarding as follows:

```bash
[bob@workstation ~]$ ssh -D 1080 bastion.securecorp.io
[bob@bastion ~]$
```

After that, the browser on Bob's workstation needs to be made SOCKS aware. In Firefox, this can be done as follows:

* Point the browser to `about:preferences`
* In the **General** tab, scroll down at the bottom, and click on **Settings...** in the **Network Settings** section
* In the opening window, choose **Manual proxy configuration**, specify **localhost** for **SOCKS Host**, **1080** as **Port** and select **SOCKS v5**
* In the same window, select **Proxy DNS when using SOCKS v5**

[firefox-socks-proxy-configuration]: img/firefox-socks-proxy-configuration.png
![SOCKS5 configuration in firefox][firefox-socks-proxy-configuration]

Bob can now point the browser to <https://sirius.securecorp.io:9090> and <https://orion.securecorp.io:9090> in order to analyze the performance problems of his two database servers using the RHEL8 web console. He can also access any other internal resources as if the browser was running on `bastion.securecorp.io`.

**Note:** Port 1080 is the [IANA registered port](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.txt) for SOCKS, but any other port can be used. The numbers in the SSH and Browser configuration have to match.

Personally I found it useful to create a separate browser profile, so it is not necessary to constantly switch between proxy configurations. A new profile can be created by passing the `-P` switch to firefox, launching the profile manager. I called my profile 'Jump'. After creating a new profile, an empty configuration is created. In this profile, I applied the configuration as described above, while leaving my default profile untouched.

[firefox-user-profile]: img/firefox-user-profile.png
![SOCKS5 configuration in firefox][firefox-user-profile]

After creating the profile, firefox can be launched as follows:

```bash
[bob@workstation ~]$ firefox -P Jump
```

Another helpful tip is to create a host specific configuration for dynamic port forwarding in your `~/.ssh/config` file, for example.

```bash
[bob@workstation ~]$ cat ~/.ssh/config
Host bastion
  Hostname bastion.securecorp.io
  User bob
  DynamicForward 1080
```

## Conclusion

There are many ways how it is possible to connecto to internal systems using a jump server, and the possibilities outlined above are by no means exhaustive. In my experience though, SSH provides you with a very powerful tool kit which in most case is available and ready to go without many hurdles. Dynamic port forwarding is one of these tools that helped me to be more productive in specific situations, so please give it a try yourself!
