Network UPS Tools (NUT) configuration files and scripts to shutdown NetApp ONTAP
================================================================================

The idea from this is to provide a graceful shutdown to NetApp ONTAP. What this setup will do:
- The NUT server will monitor the state of the UPS.
- When power is lost and UPS is on battery, start a timer for 3 minutes.
- If power comes back and UPS is back online, cancel the above timer.
- If power does not come back at the end of the 3 minutes the registered NetApp ONTAP clusters will halt.
- The NUT will shutdown when a low battery condition is reached.

Pre-requisites
--------------

A Linux server with the following tools:

* [NUT](https://networkupstools.org/)
* bash, ssh, ssh-keygen

Configuration
-------------

1. Clone this repository somewhere or in `/etc/nut`

    ```sh
    git clone https://github.com/adlytaibi/nut
    ```

2. Once the files are in `/etc/nut`, run `sshkeys` helper script as `root` to create ssh keys for nut user if keys don't already exist.

   ```sh
   # chown -R root:nut /etc/nut/
   # cd /etc/nut/
   # scripts/sshkeys 
   File "/etc/nut/scripts/ontap" is empty. Please add cluster IPs/hostnames one per line.
   ```

3. Add one or more NetApp ONTAP clusters to `/etc/nut/scripts/ontap` one per line

   ```sh
   # echo cluster1 >> /etc/nut/scripts/ontap
   ```

4. Run `sshkeys` helper script one more time to create a NUT user with ssh login and publickeys. The `sshkeys` helper script will use `admin` to login. If you use a different `admin` account to login to the cluster, edit the `useradmin` value in the script. Adding subsequent clusters to `/etc/nut/scripts/ontap` and running the `sshkeys` helper script will only add the ssh keys to the newly added cluster.

   ```sh
   useradmin="admin"
   ```

5. Make sure `upsmon.conf` is using the proper driver for your UPS system. For example: uncomment the commented block for using a USB attached APC and comment out the two block of the `dummy-ups` as shown below.

   ```sh
   #[ups]
   #  driver = usbhid-ups
   #  port = auto
   #  vendorid = 051d
   
   [ups]
     driver = dummy-ups
     port = dummy.dev
   [dummy]
     driver = dummy-ups
     port = ups@localhost
   ```

6. Restart `nut-server` and `nut-client` for the new configuration to take effect.

   ```sh
   # systemctl restart nut-server nut-client
   ```

Troubleshooting
---------------

The `dummy-ups` driver provided in this configuration works. So, before switching to use a connectivity to a real UPS in `step 5`.
Use the `dummy-ups` to test functionality of the shutdown plan on a test cluster or ONTAP simulator.
While tailing `/var/log/syslog` or similar, you can watch the timer trigger and halt ONTAP by sending events to `dummy-ups`.

* Send `on battery` event to the `dummy-ups` driver

   ```sh
   # upsrw -s ups.status=OB -u upsmon -p pass ups
   ```

   The logs show the timer is triggered for `downontap` and **will** halt the cluster(s) when the 3 minutes is up

   ```sh
   upsd[1550]: Set variable: upsmon@127.0.0.1 set ups.status on ups to OB
   upssched[4283]: Timer daemon started
   upssched[4283]: New timer: downontap (180 seconds)
   upssched[4585]: Event: downontap
   upssched-cmd: Halting ONTAP
   ```

* Send `online` event to the `dummy-ups` driver before the 3 minutes is up will cancel the timer

   ```sh
   # upsrw -s ups.status=OL -u upsmon -p pass ups
   ```

   The logs show the time being cancelled

   ```sh
   upssched[5138]: Timer daemon started
   upssched[5138]: New timer: downontap (180 seconds)
   upsd[1550]: Set variable: upsmon@127.0.0.1 set ups.status on ups to OL
   upssched[5138]: Cancelling timer: downontap
   upssched[5138]: Timer queue empty, exiting
   ```

* Send `low battery` event to the `dummy-ups` driver **will** shutdown the NUT server

   ```sh
   # upsrw -s ups.status=LB -u upsmon -p pass ups
   ```

   The logs show the FSD flag and executing a shutdown

   ```sh
   upsd[32541]: Set variable: upsmon@127.0.0.1 set ups.status on dummy to LB
   upsmon[32225]: UPS dummy@localhost battery is low
   upsd[32541]: Client upsmon@127.0.0.1 set FSD on UPS [dummy]
   upsmon[32225]: Executing automatic power-fail shutdown
   ```

License
-------

GPL

Author Information
------------------

- Adly Taibi

