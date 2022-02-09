# Slack API and PowerShell

I go through what I have learned about the Slack API and PowerShell

There is no Code here, just what I have learned

I had a task to do three things via a bot

- Setup the permmisions for the bot
- Post a message to Slack
- Get the reponse that is more than 7k
- Put the multiple reponses back together as a single JSON file

I am not going to get into how to manage Slack (mainly because I don't currently do it, but I will tekk you what needs to be done)

1. Configure the permissions for the bot
1. The slack token, this should be something like "xoxb-...". This will come from whomever enabled the channel (the same person usually who sets the permmisions)
1. Get the Channel ID: Get the details for the channel, and it is in the lower left corner of the screen, as an example: Q12P14U821A

Once you have all that, the rest is all PowerShell and REST.

First the Bot Permmisions:



oauth_config:
<br />  scopes:
<br />    bot:
- app_mentions:read
- channels:history
- channels:read
- chat:write
- chat:write.public
- commands
- incoming-webhook
- usergroups:read
- users:read
- im:history
- mpim:history
- groups:history

There are two different REST commands being used
- Chat.PostMessage (https://api.slack.com/methods/chat.postMessage)
- conversations.history (https://api.slack.com/methods/conversations.history)

The hardpart to all of this was long response and how to put them together. But let's start with the basics.

This all hinges around the invoke-RestMethod (https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod). There are a few parameters needed for this.
- Method
- Body
- ContentType
- Headers



Setup your tokens and channel

```powershell
$SlackToken = "xoxb-95..."
$SlackChannel = "#powershell-test"
$SlackChannelID = "C12P16U822Q"
```
Setup the Headers needed
```powershell
$Slackheaders = @{
    'Authorization' = 'Bearer xoxb-95...' #the same as the Token but with Bearer in front of it
    'Scope'         = 'bot'
}
```

Okay, Let's send a message I pass the command to it the function and return the output. This is all about using R4 in Slack. But the key here is that you need to know the ID of the think you want to post to, or use standard @ mentions.  Bots just work a little different so I cannot do an @R4 I actually have to use the Bot ID.

```powershell
function invoke-slackpost ($slackcmd) {
    $NameSearchTemplate = @"
    {
        "type": "app_mention",
        "channel": "$SlackChannel",
        "type": "mrkdwn",
        "token": "$SlackToken",
        "text": "$slackcmd",

    }
"@
    $restcmd = Invoke-RestMethod -Uri "https://slack.com/api/chat.postMessage" -Method Post -Body $NameSearchTemplate -ContentType 'application/json; charset=utf-8' -Headers $Slackheaders
    return $restcmd
}

$postmessage = invoke-slackpost "<@BotID> command"
```

So now I have sent the message and I want to get the response.  This is not a nested reply, it is a new message. So we will use conversations.history

First I need to get the timestamp of that message so I can get the next message after it.  I know there is a flaw in this that if somebody else posts to that channel before the reply, I will get their response. I know there is a way to get the correct message.

```powershell
$ts = $postmessage.ts`
```

So now we get the first response back.

```powershell
$firstresponse = Invoke-RestMethod -Uri "https://slack.com/api/conversations.history?channel=$SlackChannelID&inclusive=false&limit=1&oldest=$ts&pretty=1" -Method Get -Headers $Slackheaders
```

This is where it gets fun.  This message will return the following two values
- response_metadata.next_cursor
- has_more
- messages.text

Next cursor and has_more work together in building a long message (like JSON). The has_more tells you if there is more in that statment. Remember a slack response, specifically JSON is broken down into 7K (or 8K?) sizes. So a long return will have several parts to it.  The has_more tells you if there is more to that slack message. The cursor tells the unique ID of the next message.

So what you end up with is a loop that looks like the following


```powershell
$script:cursor = $firstresponse.response_metadata.next_cursor
    do {
        $secondresponse = Invoke-RestMethod -Uri "https://slack.com/api/conversations.history?channel=$SlackChannelID&inclusive=false&limit=1&oldest=$ts&pretty=1&cursor=$script:cursor" -Method Get -Headers $Slackheaders
        $tempjson = "$($secondresponse.messages.text)"
        $script:cursor = $secondresponse.response_metadata.next_cursor
        $strjson = $strjson + $tempjson
    }
    while (
        ((Invoke-RestMethod -Uri "https://slack.com/api/conversations.history?channel=$SlackChannelID&inclusive=false&limit=1&oldest=$ts&pretty=1&cursor=$script:cursor" -Method Get -Headers $Slackheaders).has_more) -like "True"
    )
```
Finally you can access the JSON File
```powershell
$json = $strjson | ConvertFrom-Json
```

This will return a JSON object something like this (just a snippit)

`
audio-test:@{branch=kong-collabos-1.5.x-adobe; data=; date_created=2022-01-24 20:12:00+00:00;manifest=10.8.671; version=10.8.671}
`

So finally let's make a menu based off the JSON file and return an object of the menu choice. Nothing too spectacular about building this menu, but wanted to show a way of how to access the data from the JSON. Even though Data shows blank in this example, it is there.

```powershell
$i = 1
    foreach ($build in $json.psobject.properties.name) {
        Write-Host "$i) $($build) ($($json.$build.data.android))"
        $i++
    
    }
    [int]$ChannelBuild = Read-Host 'Which build to use?'
    $Manifest = ($json.psobject.Properties.name[$ChannelBuild - 1])
    $Android = $json.($json.psobject.Properties.name[$ChannelBuild - 1]).data.Android
    $Branch = $json.($json.psobject.Properties.name[$ChannelBuild - 1]).Branch
    $CollabOS = $json.($json.psobject.Properties.name[$ChannelBuild - 1]).data.CollabOS
    $ChannelChoice = @{Manifest = $Manifest; Android = $Android; Branch = $Branch; CollabOS = $CollabOS }
```
