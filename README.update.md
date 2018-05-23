# Table of Contents
   * [Table of Contents](#table-of-contents)
   * [Updating](#updating)
      * [Updating Factomd](#updating-factomd)
      * [Updating Processes](#updating-processes)
         * [0. Checking Node Health](#0-checking-node-health)
            * [1. Checking DBHeight](#1-checking-dbheight)
            * [2. Check node is syncing minutes](#2-check-node-is-syncing-minutes)
            * [3. Check process list for \<nil\>s](#3-check-process-list-for-nils)
         * [1. Brain Swapping / Brain Transferring](#1-brain-swapping--brain-transferring)
            * [Process](#process)
            * [Technical Information](#technical-information)
         * [2. Updating factomd docker container](#2-updating-factomd-docker-container)





# Updating

Various updating procedures.

## Updating Factomd

When updating, factomd will have to be shut down for a brief period of time. Because of this downtime, authority servers must use brainswapping to prevent their signing identity from going offline.

There are various types of updates to the source code, and must be handled appropriately.
1. Backwards Compatible
2. Non-backwards Compatible
3. Non-backwards Compatible with activation height

All updates use the same tools/processes. Once you are familiar with the processes, it is just a matter of using the right ones in the right order

## Updating Processes

These are the various processes that an operator needs to know before performing an update. All processes will be listed here, then update instructions will have the instructions listed using the processes, rather than detailing the processes each time.

Each Process will have the process steps, as well as some additional info for additional understanding.


### 0. Checking Node Health

There are some simple checks you can do to determine your node's health. This will be from your node's perspective, so it is not a perfect check, but can be done before performing procedures to get a rough check if things are working.

1. Check your node's height against the network's height [Details](#1-checking-dbheight)
2. Check your node is syncing minutes : [Details](#2-check-node-is-syncing-minutes)
3. Check the process list for \<nil\>, that indicates some network instability : [Details](#3-check-process-list-for-s)


#### 1. Checking DBHeight

Checking your node's height against the node of the network ensures your database is synced. To get the network height you will need another node, or you can use the explorer. But keep in mind the explorer is not "live" and can lag by a block or two.

Use the `Your Block Height` on the control panel as an indicator of the height you are synced too.

#### 2. Check node is syncing minutes

Each block while it is being created is split into 10 minutes, and each minute your node uses as a syncing point.
To check if your node is following minutes, go to the control panel -> `More Detailed Node Information` -> `Summary`.
You will see something along the lines of:
```
===SummaryStart===
   FNode04[f0b7e3] L___vm01  0/ 0  0.0%  0.000   165[e0b9f8] 163/166/167  7/ 7         0/0/0/0                43400/0/0/0      0     0     2/40/100           0/0/0   0.07/0.00 0/0 - 309415
```
The `7/7` means you are on minute 7. On a follower node it will appear as `_/7`. If this number is on `0` and stuck for over a minute, your node is not syncing minutes.

#### 3. Check process list for \<nil\>s

The Process list is located in the control panel (localhost:8090) -> `more detailed node information` -> `Process List`. Simply do a `ctrl+f` and search for `<nil>`. If any appear, your node might not be syncing well with the network.

### 1. Brain Swapping / Brain Transferring

Brain swap/transfer is the simple idea of moving an identity from 1 node to another, such that the identity never appears to be offline from the perspective of the network. It is called a swap when both nodes switch with the other node's identity, and therefore 'swap' positions. It is a transfer when only 1 identity is involved.

#### Process

Definitions:
- Config File: The `factomd.conf` file, contains identity info
- Identity: The tuple(`IdentityChainID`, `LocalServerPrivKey`, `LocalServerPublicKey`) which can be placed in a config file.
- Node: A factomd process, typically the docker container, that reads from a `factomd.conf`

Prequsites:

- 2 fully synced healthy factomd nodes
    - Will be calling them N<sub>A</sub> and N<sub>B</sub>
    - Health checking can be seen [here](#0-checking-node-health)
- An Identity in  N<sub>A</sub>
    - If doing a swap, make sure all N<sub>A</sub>->N<sub>B</sub> instructions are also done N<sub>B</sub>->N<sub>A</sub>
    - An Identity to move on N<sub>A</sub>
        - An Identity is composed of the following 3 values in your factomd.conf:
            - `IdentityChainID`
            - `LocalServerPrivKey`
            - `LocalServerPublicKey`

Steps to perform a brain transfer of moving the identity on N<sub>A</sub> to be on N<sub>B</sub>. Once performed, N<sub>A</sub> can be taken offline.

1. Ensure both nodes are synced with the network.
2. Open `factomd.conf` on **both** nodes, **Do not close the files until the end of step 2**

On each file, look for the line: (if not there, append it)
```
;ChangeAcksHeight                      = 0
```

The config file uses `;` as a comment indicator, so remove the `;` character at the start of the line of both files.

```
ChangeAcksHeight                      = 0
```

The number you will specify is the height at which the brain transfer will occur. Once this height is reached, the node reading from `factomd.conf` will reload it's identity from said config file. So move the identity from N<sub>A</sub> to N<sub>B</sub> can be achieved by moving the following lines (deleting on A's conf, and pasting to B's):
```
IdentityChainID           = 8888880000000000000000000000000000000000000000000000000000000000
LocalServerPrivKey        = 4c38c72fc5cdad68f13b74674d3ffb1f3d63a112710868c9b08946553448d26d
LocalServerPublicKey      = cc1985cdfae4e32b5a454dfda8ce5e1361558482684f3367649c3ad852c8e31a
```

Now the identity should only live in 1 of the 2 configs, never both. Once it is moved, we can instruct factomd to reload it's identity at a future height. To be on the safe side, take whatever height is listed on the control panel as the current height and add 3 to it. For this example, we will say the node is currently on height 1000. In **both** configs, set the `ChangeAcksHeight` to `currentheight+3`, which in our example is 1003

```
ChangeAcksHeight                      = 1003
```

It is important to set this on both nodes, and N<sub>A</sub> needs to release it's identity at the same time N<sub>B</sub> takes it.

Once `ChangeAcksHeight` is set and the identity is moved, you can save and close the file

3. The brain transfer was initiated. Now you must wait for the block height specified to arrive (~20-30min). To verify it has worked, go to the control panel of both nodes. Click `More Detailed Node Information` -> `Servers` -> `My Node`. The Identity ChainID shown should be your identity on N<sub>B</sub> and gibberish on N<sub>A</sub>.

4. At this point your identity is on N<sub>B</sub>, and you can do as you wish with N<sub>A</sub> (update, turn offline, decomission, etc).

#### Technical Information

Understanding the brainswap could be beneficial when performing one. The first thing to understand is that all factomd's are the same. The follower runs the same code as a Federated server. Even followers have identities, it's just that their identities are effecitivly gibberish. The blockchain dictates the authority set, and every factomd keeps an eye on the authority set, and will perform authority actions if it's identity is in the set.

When you reload the identity of a factomd (change the identity to something else), it checks if it is in the authority set, and behaves appropriately. This means when N<sub>B</sub> hits the ChangeAcksHeight specified, it see's that is is now an authority, and begins performing the role. N<sub>A</sub> see's it is no longer a leader, and begins to act as a follower.

If N<sub>A</sub> and N<sub>B</sub> are both the same identity, they will both try to perform as an authority, however the network does not allow 2 nodes to share an identity. Both N<sub>A</sub> and N<sub>B</sub> will detect that aswell, and commit seppuku by crashing. This does mean that if you fail the brainswap, your nodes will go offline.

From the network point of view, an identity is online as long as it continues to sign messags on the network. By doing a brainswap on a block boundary, N<sub>A</sub> will sign messages for the identity at the  `< height`, and N<sub>B</sub> will sign all messages `> height`. From a network point of view, the identity is continously signing messages, and never goes offline. This does strengthen the notion that an identity and node are not necessarily the same thing.


### 2. Updating factomd docker container

When updating the factomd docker container you will need to know the `docker run ...` command used to start the container. The default run command for mainnet can be found [here](README.md#from-the-docker-cli-recommended-and-better-tested).

The simpliest way to update, is to stop factomd, remove the container, then choose the new image when using `docker run...`

1. Stop factomd container : `docker stop factomd`
2. Remove the container  : `docker rm factomd`
3. Start the new, updated image : `docker run ..dockerflags.. factominc/factomd:v#.#.#-alpine ..factomdflags..` replacing `#.#.#` with the newest version.
