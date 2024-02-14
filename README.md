# Farcaster Hubble/Replicator Install Guide

Any problems, please ping me on Farcaster at [https://warpcast.com/agadoo](https://warpcast.com/agadoo)

**_WARNING_** - This guide is purely for development purposes (no idea if suitable for production).

This method assumes you want to install hubble/replicator on a server (if using on a local machine/desktop/laptop, you may also need to look into port forwarding to get this to work with your router).

**_Reminder_** - this is only relevant if you want to install the replicator (to save data into a postgres db). If you just want to install hubble, run a node and query things via the httpapi, you can ignore this guide (and follow the [Farcaster hubble install documention](https://docs.farcaster.xyz/hubble/install))

And fyi, am a complete docker/linux/shell script left curver, so please let me know what should be changed below :)

### Issue

Installing hubble and replicator so they can work together locally is not trivial (most likely due to issues with HUB_HOST and the likely error "Unhandled promise rejection: Error: Unable to communicate with xyz:2283).

The reason for this - when they are installed, the hubble and replicator are in different docker containers (using different docker networks). This means that out of the box, they can't 'see' each other locally.

### Solution 1 (external IP - works but not ideal)

1. Install the [hubble](https://docs.farcaster.xyz/hubble/install) and [replicator](https://docs.farcaster.xyz/developers/guides/apps/replicate) as per the Farcaster documentation.
2. During the replicator setup, when asked for 'HUB*HOST' you can use the server's external IP with port 2283.
   (\_e.g. 12.34.56.78:2283*)

This solves the issue of different docker containers/networks by going via the network stack (i.e. effectively going out to the internet and back again). Whilst this should work, it loses the benefits of running things locally (most notably increased latency).

### Solution 2 (hubble/replicator on same docker network)

1. Install the [hubble](https://docs.farcaster.xyz/hubble/install) and [replicator](https://docs.farcaster.xyz/developers/guides/apps/replicate) as per the Farcaster documentation.

2. During the replicator setup, when asked for 'HUB_HOST', enter hubble_my-network:2283 (hubble_my-network is the default docker network set up when installing hubble).

3. You will eventually see a log error (something like 'ERROR - Unhandled promise rejection: Error: Unable to communicate with xyz:2283 ) - when you see this, can ctrl+c to quit.

4. Now open the replicator docker-compose file (depends on the install location e.g. nano /home/yourusername/replicator/docker-compose.yml). For those not familiar with docker-compose files - indentation and spaces matter! Anything indented underneath another line below should be two spaces exactly.

5. Around line 27, find

```
networks:
  - replicator-network
```

and replace with

```
networks:
   - replicator-network
   - hubble_my-network
```

6. Around line 96, find

```
networks:
    replicator-network:
```

and replace with

```
networks:
  replicator-network:
  hubble_my-network:
    external: true
```

7. By adding the default hubble network (hubble_my-network) to the replicator using 'external: true', it should connect to both it's own network (replicator_replicator-network) and the hubble's network.

8. Save this and now restart the replicator (navigate to the /replicator folder and run docker compose up)

9. You should see no similar ERROR message as above, and eventually see log messages coming in like 'INFO (7): Enqueuing jobs for backfilling FID registrations...'. This means it has worked. If you need to run other commands, you can run ctrl+c to quit, then restart the replicator without logging everything to the console

```
docker compose up -d
```

10. To verify things are working, from within the /replicator folder you can open postgres using:

```
docker compose exec postgres psql -U replicator replicator

```

11. Run a quick query - e.g.

```
select * from fids limit 10;
select * from casts limit 10;
```

If you see any data at all, this means the replicator is working correctly.

### Extra tip - custom command to open postgres

Remembering the command to start postgres is a pain, so here's how to add a custom command to handle this.

1. In the following code, MYCOMMAND is the command you want to use (can be anything as long as it is one word/string with no spaces). FOLDER_LOCATION is the location of your replicator installation.
   Once you have these, in your server console type:

```
echo "alias MYCOMMAND='cd /FOLDER_LOCATION/replicator && sudo docker compose exec postgres psql -U replicator replicator'" >> ~/.bashrc
```

```
e.g. for me, it might be

echo "alias hubble-db='cd /home/agadoo/replicator && sudo docker compose exec postgres psql -U replicator replicator'" >> ~/.bashrc
```

2. Run the following to make the changes

```
source ~/.bashrc
```

3. You should now be able to just type MYCOMMAND (e.g. for me hubble-db) from anywhere and the postgres terminal should open

### Default INFO

In case it is of use, default setups will result in the following:

#### Hubble

Network in docker-compose: my-network
Docker network: hubble_my-network (docker auto-adds the process name as a prefix)
Docker image: farcasterxyz/hubble:latest
Docker container name: hubble-hubble-1

#### Replicator

Network in docker-compose: replicator-network
Docker network: replicator_replicator-network (docker auto-adds the process name as a prefix)
Docker image: farcasterxyz/replicator:latest
Docker container name: replicator-replicator-1
