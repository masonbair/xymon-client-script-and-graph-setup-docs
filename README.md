# Setting Up Custom Scripts and Graphs in Xymon

This guide walks through setting up a machine with a custom script and custom graph in Xymon.

## Useful Links

If you need help or get stuck:
- https://xymon.sourceforge.io/xymon/help/xymon-tips.html#scripts
- https://xymon.sourceforge.io/xymon/help/howtograph.html

---

## Client Setup

The client machine used in this situation was an Ubuntu 24.04 server, and the xymon-client was installed via:

```bash
apt install xymon-client
```

- The apt install also asks for an IP address, which you can find by running: `nslookup megatron.cs.kent.edu`

### Creating the Custom Script

When working with a custom script, you need to first create the script in this folder:

```
/usr/lib/xymon/client/ext
```

- This is the folder for scripts that the client runs and sends data to the server.
- The script needs to be formatted in a specific way. You can see an example script here: https://xymon.sourceforge.io/xymon/help/xymon-tips.html#scripts

**Example:** For the first use case, I needed to set up a script to keep track of GPU data, so I made a script called `gpu_monitor.sh` and gave it the necessary permissions:

- Xymon has its own user, so chown the script for xymon:
  ```bash
  chown xymon {scriptname}.sh
  ```

- Xymon also needs access to execute it:
  ```bash
  chmod 700 {scriptname}.sh
  ```

### Configuring the Client to Run the Script

After creating the script and giving it permissions, you need to tell Xymon to see it and run it. Next, go to:

```
/etc/xymon/clientlaunch.cfg
```

In this file, you will already see some basic configuration, but we need to add a new section. This is what the section looks like for the GPU monitor:

```ini
[gpu_monitor]
        ENVFILE $XYMONCLIENTHOME/etc/xymonclient.cfg
        CMD $XYMONCLIENTHOME/ext/gpu_monitor.sh
        LOGFILE $XYMONCLIENTLOGS/gpu_monitor.log
        INTERVAL 5m
```

Once you add your script to the file, make sure to restart the xymon-client to see the new script:

```bash
systemctl restart xymon-client
```

Then, after the amount of time set as the interval, you should see the xymon-server sees the column.

---

## Server Graph Setup

Now this is where things get more complex. For the graph, the xymon-server reads data from a `.rrd` file for figuring out what to display on the graph. So you may need to modify your script to send over data for the xymon-server to read. In this case, I followed the NCV (Name-Colon-Value) method as talked about here: https://xymon.sourceforge.io/xymon/help/howtograph.html

This is what mine looked like, and this is part of my script, located at the end:

```bash
# This data in MSG allows for the server to read via the NCV method
   MSG="${MSG}
gpuutil: ${GPU_UTIL}
memutil: ${GPU_MEM_UTIL}
temp: ${GPU_TEMP}
power: ${GPU_POWER}
"
```

### Verifying Data Collection

If your script is all good with the data sending, next comes the graph setup bit. This process is strictly on the xymon server, since the xymon server does the graphical displays.

To make sure the server is receiving the data, go check the xymon server RRD folder. This is where Xymon stores all the collected data, located here:

```
/var/lib/xymon/rrd/{client_hostname}
```

If you see the name of the column followed by `.rrd` (e.g., `gpu.rrd`), you can view it to see if the right data is collected via:

```bash
rrdtool info {filename}.rrd
```

### Configuring the Server

Once you've verified the data, go to the `/etc/xymon` directory and edit `xymonserver.cfg`.

In this file, the main areas of interest are: `TEST2RRD` and `GRAPHS`.

If you are using the NCV method for the data, then at the end of the `TEST2RRD` definition, add your configuration. In my case, at the end I added:

```
gpu=ncv
```

Then in the `GRAPHS` section, I added to the end:

```
gpu
```

The final step in this file is to make an NCV variables section to tell Xymon what variables to pull from the client data. This can just be added as an extra configuration line after the `GRAPHS` declaration.

**Example:**

```
NCV_gpu="gpuutil:GAUGE,memutil:GAUGE,temp:GAUGE,power:GAUGE"
```

- `GAUGE` is the variable type, which means positive and negative values, non-incremental

### Creating the Graph Definition

The final file you need to edit:

For graphs, you'll need to make a new `.cfg` file in this directory:

```
/etc/xymon/graphs.d
```

From there, this is what my GPU graph looks like:

```
# Mason Bair's GPU Utilization Graphing section
[gpu]
      TITLE GPU 0 Utilization and Temperature
      YAXIS % / Celsius
      DEF:gpu_util=gpu.rrd:gpuutil:AVERAGE
      DEF:mem_util=gpu.rrd:memutil:AVERAGE
      DEF:temp=gpu.rrd:temp:AVERAGE
      DEF:power=gpu.rrd:power:AVERAGE
      LINE2:gpu_util#00CC00:GPU Utilization
      GPRINT:gpu_util:LAST: \: %5.1lf (cur)
      GPRINT:gpu_util:AVERAGE: \: %5.1lf (avg)
      GPRINT:gpu_util:MAX: \: %5.1lf (max)\n
      LINE2:mem_util#0000FF:Memory Utilization
      GPRINT:mem_util:LAST: \: %5.1lf (cur)
      GPRINT:mem_util:AVERAGE: \: %5.1lf (avg)
      GPRINT:mem_util:MAX: \: %5.1lf (max)\n
      LINE2:temp#FF0000:Temperature
      GPRINT:temp:LAST: \: %5.1lf C (cur)
      GPRINT:temp:AVERAGE: \: %5.1lf C (avg)
      GPRINT:temp:MAX: \: %5.1lf C (max)\n
      LINE2:power#ebcc1c:Power
      GPRINT:power:LAST: \: %5.1lf (cur)
      GPRINT:power:AVERAGE: \: %5.1lf (avg)
      GPRINT:power:MAX: \: %5.1lf (max)\n
```

This graph has a `TITLE` and a `YAXIS` that shows the value type being displayed. Then you define some variables with `DEF`, which you pull from the RRD file that was created. In addition, we have sections of `LINE2` followed by three `GPRINT` statements. This tells the graph what to display.

Finally to finalize things do a:

```bash
systemctl restart xymon
```

---

## Summary

That's it! Your custom script should now be collecting data on the client and displaying graphs on the Xymon server.
