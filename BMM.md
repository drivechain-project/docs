# How to blind merge mine (BMM) sidechain blocks

### 1. Setup configuration files for DriveNet and the sidechain you are mining. 

Make the rpcuser and rpcpassword whatever you want, but they must be the same
for DriveNet and the sidechain(s).

DriveNet (~/.drivenet/drivenet.conf):

    rpcuser=drivenet
    rpcpassword=drivechain
    server=1

Sidechain One (~/.sidechain/sidechain.conf):

    rpcuser=drivenet
    rpcpassword=drivechain
    
Payment sidechain (~/.payment/payment.conf):

    rpcuser=drivenet
    rpcpassword=drivechain

### 2. Start the sidechain(s).

    ./sidechain-qt
    ./payment-qt

If you are going to use the automated BMM page, you must start DriveNet with the
--minersleep option set. Set this value higher than the automated bmm refresh
interval. For example, if the refresh interval is set to 1 second, set
--minersleep=1500 (1.5 seconds).

### 3. Start DriveNet with command line options

    ./drivenet-qt --minersleep=1500

 ### 4. Enable automated BMM (check the checkbox)

Turn on automated BMM, and set the refresh interval to anything faster than
what --minersleep has been set to.
