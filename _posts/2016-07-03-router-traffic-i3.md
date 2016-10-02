---
layout: post
title: Measuring router traffic and displaying it in i3
---

## Motivation

At the dorm I'm currently living in, there's a limit on how much network traffic you are allowed to generate. More specifically, if you surpass a set amount of up/down traffic within 24 hours, you will be temporarily blocked off the Internet. The important thing to notice here is that the traffic counter does not reset at a specific time, but always counts the traffic used in the last 24 hours.

To avoid surpassing this limit, it would be useful to have som kind of indication on how much traffic you have currently generated. To achieve this, we'll use OpenWrt and vnstat to collect traffic data on the router, some Python scripting to retrieve the data, and i3bar to display it to the user.


## Setup

##### OpenWrt
The first step will be to install OpenWrt, which is a Linux-based operating system for network routers. OpenWrt includes its own package manager, which makes it easy to install new programs (like vnstat) on it. It's also a nice OS in general, in my opinion.

To intall it, you first need to find out your router model and download the corresponding image file from OpenWrt's website. Use [this page](https://wiki.openwrt.org/toh/start) to find your router, and follow the guides on the wiki for flashing/installing it on your router and setting it up. If you router is not listed, it is unfortunately not supported.

You'll also have to install your SSH key as an authorized key on the router, because otherwise you'll have to manually provide a password each time our data polling script (described later) is being run.

#### vnstat
When that is all done, it's time to install vnstat, which is a program for measuring network traffic. To install it we'll use OpenWrt's own package manager, so just SSH into your router and run the following commands:

```
opkg update
opkg install vnstat
```

You then need to find the name of the interface you want to monitor. The one we want to use here is the WAN interface, as that is what the traffic counter checks. The easiest way to find the WAN interface name (as far as I know) is to just login to the router through the web browser GUI (<http://192.168.1.1>) and check the name of the WAN interface under the Network->Interfaces tab. For me, the WAN interface is called eth1.

![OpenWrt network interfaces tab](https://i.imgur.com/lpL5J1c.png)

We then create a database for vnstat, and enable the vnstatd daemon to perform monitoring in the background:

```
vnstat -u -i eth1

/etc/init.d/vnstat enable
/etc/init.d/vnstat start
```

If everything went as it should, you should see some traffic data with the command `vnstat -i eth1 -h` after a little while. You may need to wait an hour or two before the `-h` option (`--hours`) - which we are going to use here - will give you data as output.

## Collecting and parsing the data

#### The data format
After a while, `vnstat -i eth1 -h` will print something like this:

```
 eth1                                                                     20:04
  ^                                                                    r
  |                                                     r     r        r
  |                                                     r     r        r
  |                                                     r     r        r
  |                                                     r     r        r
  |                                                     r     r        r
  |                          r                          r     r  r  r  r
  |                          r                          r     r  r  r  r
  |                          r                          r     r  r  r  r
  |                          r                          r     r  r  r  r
 -+--------------------------------------------------------------------------->
  |  21 22 23 00 01 02 03 04 05 06 07 08 09 10 11 12 13 14 15 16 17 18 19 20

 h  rx (KiB)   tx (KiB)      h  rx (KiB)   tx (KiB)      h  rx (KiB)   tx (KiB)
21       4366         16    05     158855       6286    13       4682         18
22       4201         16    06       4323         17    14     304711      13718
23       4447         17    07       4360         17    15      24004       1642
00       4533         15    08       4628         18    16     303093      11844
01       4581         16    09       4511         16    17     145010       6037
02       4351         16    10       4629         18    18     138638       5823
03       4642         16    11       4292         17    19     323162      11121
04       4166         16    12       4582         18    20       8392        372
```

What we want is the sum of all the rx entries (the data the router receives from the WAN, e.g. when you download something) and tx entries (the data the router sends to the WAN, e.g. when you upload something). You may also have noticed that vnstat has two other options - `--days` and `--short` - which appear to do what we want without us having to sum those numbers. Unfortunately, these options give the daily data usage (resetted at midnight), and not the immediate last 24 hours as we want.

There are lots of ways to do this summing programmatically, but here we'll do it with a Python (3) script.

#### Writing the script
First we'll write some code for collecting the raw data:
{% highlight python %}
update_cmd = ['ssh', 'root@192.168.1.1', 'vnstat -i eth1 -u']
retrieve_cmd = ['ssh', 'root@192.168.1.1', 'vnstat -i eth1 -h']

subprocess.call(update_cmd)
raw_output = subprocess.check_output(retrieve_cmd).decode('utf-8')
{% endhighlight %}

Here we first run vnstat with the `-u` option, to update the database before we collect the data. The database updates at 30 minute intervals anyways (can be changed by editing the `/etc/vnstat.conf` file), but it's nice to get the most recent data.

To sum the values, we'll first strip away the leading graph and value header by only saving the data after the last parenthesis in the output.

{% highlight python %}
output = raw_output[raw_output.rfind(')')+1:]
{% endhighlight %}

The output variable will then look something like this:

```
21       4366         16    05     158855       6286    13       4682         18
22       4201         16    06       4323         17    14     304711      13718
23       4447         17    07       4360         17    15      24004       1642
00       4533         15    08       4628         18    16     303093      11844
01       4581         16    09       4511         16    17     145010       6037
02       4351         16    10       4629         18    18     138638       5823
03       4642         16    11       4292         17    19     323162      11121
04       4166         16    12       4582         18    20       8392        372
```

If we read each value from left to right, top to bottom, we can see that the sum of the rx values will be the sum of every third number, starting from the second number (in this case: 4366). For tx it will be the same, just starting from the third number (16) instead. This can be done in the following way with Python:

{% highlight python %}
output = output.split()
rx_sum = sum(int(output[i]) for i in range(1, len(output), 3))
tx_sum = sum(int(output[i]) for i in range(2, len(output), 3))
{% endhighlight %}

The collected data is in [KiB](https://en.wikipedia.org/wiki/Kibibyte), but I want it in GiB instead. To achieve this, we just divide the values by 1024^2:

{% highlight python %}
rx_sum /= 1024**2
tx_sum /= 1024**2
{% endhighlight %}

If you for some reason want the values in GB instead, you can use this formula (after the KiB to GiB conversion) for GiB to GB conversion:

{% highlight python %}
gb = gib * (2**30 / 10**9)
{% endhighlight %}

We're almost done now. All that is left is to print the values in a nice way. I want the textual output to be in the format 'rx / tx' with two decimal places, which can be done the following way with Python's format strings:

{% highlight python %}
print('{:.2f} / {:.2f}'.format(rx_sum, tx_sum))
{% endhighlight %}

The complete script, slightly tidied up by placing the code in functions, can be found in the appendix below.

## Displaying the data in i3bar

Having written the script in the previous section, we can now just run it to get the desired data. It is still a bit cumbersome though, as we need to manually run the script in a terminal each time we want to see the output. To constantly show updated data to the user, without any user interaction, we'll integrate the script into i3bar, which is the status bar for the i3 window manager. This can of course be skipped if you don't use i3, or want to use the data in some other way.

To show the script output in i3bar, we'll use a program called i3blocks. This program basically runs specified scripts at specified intervals, and sends their outputs to i3bar for display. The reason for using i3blocks instead of the standard i3status, is that i3status has poor support for custom scripts.

First install i3blocks with your package manager (`apt-get install i3blocks`, `pacman -S i3blocks`). When that's done, change your i3 config to use i3blocks as i3bar's status command:

```
bar {
    status_command i3blocks
}
```

Then make a file named `.i3blocks.conf` in your home directory and add the following lines to it:

```
[router traffic]
label=Traffic:
command=/path/to/script/router_traffic.py
interval=600

# If you also want to show the current date & time, add this too
[datetime]
command=date '+%F  %T'
interval=1
```

Finally, restart i3 ($mod+shift+r or `i3-msg restart`).

You of course have to change the path to where the script file is located. We set the interval to 600 seconds, so that it polls data every 10th minute. Also, if this is the only entry you have in your i3blocks config, it will be a bit lacking. To customize it further, you should consult i3block's documentation (`man i3blocks` is pretty helpful here).

## Conclusion

If you've followed through this whole article, you'll now see the total amount of traffic generated in the last 24 hours in your i3bar, polled from your router running OpenWrt and vnstat. Using the config file in the previous section, it may look something like this:

![i3bar with traffic data](https://i.imgur.com/W5rKJl8.png)

Which indicates that 1.31 GiB has been downloaded and 0.05 GiB has been uploaded in the last 24 hours.

## Appendix

{% highlight plaintext %}
router_traffic.py
{% endhighlight %}

{% highlight python %}
#!/bin/env python3

import subprocess


def get_raw_output():
    update_cmd = ['ssh', 'root@192.168.1.1', 'vnstat -i eth1 -u']
    retrieve_cmd = ['ssh', 'root@192.168.1.1', 'vnstat -i eth1 -h']

    subprocess.call(update_cmd)
    return subprocess.check_output(retrieve_cmd).decode('utf-8')


def calculate_traffic_sums(raw_output):
    output = raw_output[raw_output.rfind(')')+1:].split()

    rx_sum = sum(int(output[i]) for i in range(1, len(output), 3))
    tx_sum = sum(int(output[i]) for i in range(2, len(output), 3))

    return convert_to_gib(rx_sum), convert_to_gib(tx_sum)


def convert_to_gib(kib_value):
    return kib_value / (1024**2)


if __name__ == '__main__':
    raw_output = get_raw_output()
    sums = calculate_traffic_sums(raw_output)
    print('{:.2f} / {:.2f}'.format(sums[0], sums[1]))
{% endhighlight %}
