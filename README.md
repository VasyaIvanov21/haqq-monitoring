# haqq-monitoring 
#### written by the CryptoBox validator as part of the testnet task 

# Setting up alerts in TG

##### First you need to register a bot and get its id and token.There is a special bot for this >>>  https://telegram.me/botfather
##### Write to him  "/start". Then the command to create your bot "/newbot"
##### Then come up with a name for your bot and write it. If it is already in use, the bot will ask you to come up with another name.
##### If successful, you will see information with your token data for accessing the HTTP API
##### Write down the token data, they will be useful to us later.Visit your bot and click start
##### Now we need to find your telegram account ID
##### Visit the bot and click start. https://telegram.me/getmyid_bot
##### Great, now we just need to set up a small script.

```
mkdir $HOME/Haqq_alert_TG
```
```
sudo apt install nano
nano $HOME/Haqq_alert_TG/Haqq_alert_TG.sh
```
#### Substitute your data in the script in !!!!! TG_BOT=(Telegram bot API) and TG_ID=(Telegram ID)
Also specify the name of your validator NODE_NAME="".
If necessary, change the ports of your node.

```
#!/bin/bash

# Node name, e.g. "Cosmos"
NODE_NAME=""
# File name for saving parameters, e.g. "cosmos.log"
LOG_FILE="/root/Haqq_alert_TG/Haqq.log"
# Your node RPC address, e.g. "http://127.0.0.1:26657"
NODE_RPC="http://127.0.0.1:26657"
# Trusted node RPC address, e.g. "https://rpc.cosmos.network:26657"
SIDE_RPC="https://haqq-t.rpc.manticore.team:443"
# Telegram bot API
TG_BOT=
# Telegram chat ID
TG_ID=
ip=$(wget -qO- eth0.me)

touch $LOG_FILE
REAL_BLOCK=$(curl -s "$SIDE_RPC/status" | jq '.result.sync_info.latest_block_height' | xargs )
STATUS=$(curl -s "$NODE_RPC/status")
CATCHING_UP=$(echo $STATUS | jq '.result.sync_info.catching_up')
LATEST_BLOCK=$(echo $STATUS | jq '.result.sync_info.latest_block_height' | xargs )
VOTING_POWER=$(echo $STATUS | jq '.result.validator_info.voting_power' | xargs )
ADDRESS=$(echo $STATUS | jq '.result.validator_info.address' | xargs )
source $LOG_FILE

echo 'LAST_BLOCK="'"$LATEST_BLOCK"'"' > $LOG_FILE
echo 'LAST_POWER="'"$VOTING_POWER"'"' >> $LOG_FILE

curl -s "$NODE_RPC/status"> /dev/null
if [[ $? -ne 0 ]]; then
    MSG="$ip  узел остановлен "
    MSG="$NODE_NAME $MSG"
    SEND=$(curl -s -X POST -H "Content-Type:multipart/form-data" "https://api.telegram.org/bot$TG_BOT/sendMessage?chat_id=$TG_ID&text=$MSG"); exit 1
fi


if [[ $LAST_POWER -ne $VOTING_POWER ]]; then
    DIFF=$(($VOTING_POWER - $LAST_POWER))
    if [[ $DIFF -gt 0 ]]; then
        DIFF="%2B$DIFF"
    fi
    MSG="$ip  размер стейка валидатора изменился  $DIFF%0A($LAST_POWER -> $VOTING_POWER)"
fi

if [[ $LAST_BLOCK -ge $LATEST_BLOCK ]]; then

    MSG="$ip узел застрял в блоке номер >> $LATEST_BLOCK"
fi

if [[ $VOTING_POWER -lt 1 ]]; then
    MSG="$ip  валидатор не активен ( выпал из актив сета) или находится в тюрьме."
fi

if [[ $CATCHING_UP = "true" ]]; then
    MSG="$ip узел находится в процессе синхронизации . $LATEST_BLOCK -> $REAL_BLOCK"
fi

if [[ $REAL_BLOCK -eq 0 ]]; then
    MSG="$ip пропало соединения с рпц нодой $SIDE_RPC"
fi

if [[ $MSG != "" ]]; then
    MSG="$NODE_NAME $MSG"
    SEND=$(curl -s -X POST -H "Content-Type:multipart/form-data" "https://api.telegram.org/bot$TG_BOT/sendMessage?chat_id=$TG_ID&text=$MSG")
fi

```


```
sudo chmod 744 $HOME/Haqq_alert_TG/Haqq_alert_TG.sh
```

#### Let's just put the script in Cron
```
 crontab -e
 ```
 
 #### We will make the script run every 5 minutes. Add this line at the end
  ```
 */5 * * * *  /bin/bash $HOME/Haqq_alert_TG/Haqq_alert_TG.sh
  ```
