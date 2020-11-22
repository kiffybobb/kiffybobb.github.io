---
title: Netcheck
author: Robert Ely
---


Our home internet connection started to seem a bit shakey recently, and with the COVID-19 pandemic sweeping Australia, I was stuck inside needing internet and not getting the results from my NBN connectivity that I would have preferred. Enter the [Raspberry Pi](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/). 

A Raspberry Pi, for those that may not know, is just a tiny, credit-card sized computer. I purchased one, and a bit of kit to go with it, and got on my way. The advantage of using such a device was that I could leave it running a script in the background that could collect data for me, allowing me to come and go as I pleased, and viewing the data later. 

## Accessing the Pi
After initially setting up the Raspberry Pi, it is possible to access it via ```ssh``` using my laptop. To do this, I first had to find it's initial IP address, which I think gave the alias of **pi@raspberrypi**. Using the Windows command line, I can ssh into the pi as follows:
```
ssh pi@raspberrypi
```
I can then access and edit files on the raspberry pi from the comfort of my laptop. 

## Checking internet connectivity
The second part of my problem was how to test how well my internet transfer speeds are performing. I perhaps could have put some effort into learning how to do this, but a tool already exists. That tool is [https://www.speedtest.net/apps/cli](speedtest-cli), brought to you buy the (assumingly) good people at [https://www.speedtest.net/](speedtest.net). It is a command-line tool that works in a similar way to the Speedtest website. It spits out some stats and tells you how quick (or slow) your internet transfer speeds are. 

## Combining the pi, pyhton and the speedtest-cli
Now I was getting somewhere. It was time to write a python script that could run the speedtest-cli on the raspberry pi and write the information to a data file (in this case CSV). It took me a few goes at getting this to work, but it ended up something like the following:

```python
# Libraries
import requests     # for checking connections
import subprocess   # for running processes external to the script, and in the background
import re           # regex
import csv          # writing things to CSV
from datetime import datetime # for logging purposes
import logging      # actually for logging purposes


# Functions
def get_connection_status(url, timeout):
    """Given a URL and timeout speed, check to see if user is connected."""
    try:
        request = requests.get(url, timeout=timeout)
        return True
    except(requests.ConnectionError, requests.Timeout) as exception:
        print(exception)
        return False


def check_speed_results():
	"""Get speedtest results by running speedtest-cli and returning shell output"""
    result = subprocess.run(['speedtest-cli'], stdout=subprocess.PIPE)
    return result.stdout.splitlines()


def get_network_and_signal():
    # get iwconfig results
    result = subprocess.check_output('/sbin/iwconfig')
    result = result.decode('utf-8').splitlines()

    # get signal
    signal = result[5]
    signal = str(signal).split("=")[1]
    signal = signal.split("/")
    signal_nom = signal[0]
    signal_dom = signal[1].split("S")[0].strip()
    signal_percent = round(((int(signal_nom)/int(signal_dom))*100),2)

    # get network
    network = str(result[0]).split(':')[1].split("\"")[1]

    return network, str(signal_percent)


def split_list(my_list):
    half = len(my_list) // 2
    return my_list[:half], my_list[half:]


def write_data_to_csv(data, filepath):
    with open(filepath, "a", newline="") as doc:
        writer = csv.writer(doc)
        writer.writerow(data)

# Main script
def main():
    begin_time = datetime.now()

    # set variables to defaults
    url = "https://www.speedtest.net"
    timeout = 5
    filename = "/home/pi/Python/netcheck/netcheck_data.csv"
    log_filename = "/home/pi/Python/netcheck/netcheck_log.log"

    download_speed = "0"
    upload_speed = "0"
    distance = "NA"
    ping = "NA"
    signal_strength = "0"
    network = "NA"

    # Get datetime variables
    now = datetime.now()
    date = now.strftime("%Y-%m-%d")
    time = now.strftime("%H:%M:%S")
    my_datetime = date + " " + time

    #check connection
    is_connected = get_connection_status(url, timeout)

    # Welcome statement
    print("=========================================")
    print("netcheck - check your internet connection")
    print("=========================================")
    print("")

    if is_connected:
        print("You are connected to the internet! Commencing speed test!")

        network, signal_strength = get_network_and_signal()

        results = check_speed_results()
        download_speed = '.'.join(re.findall(r'\d+', str(results[6])))
        upload_speed = '.'.join(re.findall(r'\d+', str(results[8])))
        distance_ping = split_list(re.findall(r'\d+', str(results[4])))
        distance = '.'.join(distance_ping[0])
        ping = '.'.join(distance_ping[1])

        data = [my_datetime, date, time, download_speed, upload_speed, distance,
                            ping, network, signal_strength]

    else:
        print("You are not connected to the internet! Please check your connection.")
        data = [my_datetime, date, time, download_speed, upload_speed, distance,
                            ping, network, signal_strength]

    # write to csv
    write_data_to_csv(data, filename)

    # end time
    execution_time = datetime.now() - begin_time

    # write to log

    logging.basicConfig(filename=log_filename, level=logging.INFO)
    logging.info('========================')
    logging.info('date: {}'.format(date))
    logging.info('time: {}'.format(time))
    logging.info('---------------------')
    logging.info('Connected: {}'.format(is_connected))
    logging.info('Download: {} Mb/s'.format(download_speed))
    logging.info('Upload: {} Mb/s'.format(upload_speed))
    logging.info('Distance to server: {} km'.format(distance))
    logging.info('Ping: {} ms'.format(ping))
    logging.info('Network name: {}'.format(network))
    logging.info('Signal Strength: {}%'.format(signal_strength))
    logging.info('Executed in: {}'.format(execution_time))
    logging.info('========================')
    logging.info('\n')


if __name__ == "__main__":
    main()

```

All the above script is basically doing is making sure there is an internet connection, then running speedtest-cli and copying information from the shell output. Then I have to trawl through the output and save the correct little chunks of information I want into some variables I have created. Then I write those little chunks of information (like download speed, upload speed, distance to sever etc.) to a CSV file for later scrutiny. 

## Running the script automatically
To run the script in the background automatically, I needed to create a **cronjob**, which is an automated task in a UNIX environment (the operating system used by the raspberry pi). From the command line I can run:
```bash
crontab -e
```
This allows me to edit the crontable(?). Then it is just a matter of adding the following:

```bash
8,38 * * * * /usr/bin/python3 /home/pi/Python/netcheck/netcheck.py > /home/pi/logs/cronlog.log 2>&1
```
This tells the pi to:
- run everything at any minute ending in *8* or *38*. 
- Use the program python3 (the python installation) from the specified path
- Run the script at the specified path (netcheck.py)
- Save a log to the specified file and path (cronlog.log)

So every half hour the raspberry runs the ```netcheck.py``` script. Laughing. 

## Visualising the data with Tableau Public
I previously had a copy of Tableau, but it expired, so I tried my luck with Tableau Public. 
I came up with the following, which you can access and play with [here](https://public.tableau.com/profile/rob8334#!/vizhome/netcheck2_1/netcheck2_1).

![tableau_screenshot.jpg]({{site.baseurl}}/img/tableau_screenshot.jpg)

The people at my ISP told me the minimum acceptable download speed was 9 Mbps. You can see from the Tableau dashboard that I am only getting speeds beyond this about 50% of the time, given **1,811** observations, or approximately 37 days. 

NOT GREAT. 

Now I just need to get the ISP to do something about it, or perhaps stop using the microwave whilst trying to stream Netflix...
