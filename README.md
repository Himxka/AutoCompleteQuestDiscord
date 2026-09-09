# 🎯 Comment utiliser le script

Ce script permet d'automatiser la progression de certaines **quêtes Discord** afin de pouvoir récupérer les orbs

> ⚠️ **Attention :** ce script utilise des fonctions internes de Discord et peut ne plus fonctionner après une mise à jour de Discord. Son utilisation peut également être contraire aux conditions d'utilisation de Discord. Utilisez-le à vos propres risques.

---

## 📋 Prérequis

* L'application **Discord Desktop** pour certaines quêtes.
* Une quête disponible dans l'onglet **Quêtes**.
* L'accès aux **DevTools** de Discord.

---

## 🚀 Utilisation

### 1. Accepter une quête

Rendez-vous dans l'onglet **Quêtes** de Discord et acceptez la quête que vous souhaitez effectuer.

### 2. Ouvrir les outils de développement

Appuyez sur :

```text
Ctrl + Shift + I
```

Cela ouvre les **DevTools** de Discord.

### 3. Ouvrir la console

Dans les DevTools, sélectionnez l'onglet :

```text
Console
```

### 4. Coller le script

Copiez l'intégralité du script ci-dessous, collez-le dans la console puis appuyez sur **Entrée**.

```js
delete window.$;
let wpRequire = webpackChunkdiscord_app.push([[Symbol()], {}, r => r]);
webpackChunkdiscord_app.pop();

let ApplicationStreamingStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getStreamerActiveStreamMetadata).exports.A;
let RunningGameStore = Object.values(wpRequire.c).find(x => x?.exports?.Ay?.getRunningGames).exports.Ay;
let QuestsStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getQuest).exports.A;
let ChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getAllThreadsForParent).exports.A;
let GuildChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.Ay?.getSFWDefaultChannel).exports.Ay;
let FluxDispatcher = Object.values(wpRequire.c).find(x => x?.exports?.h?.__proto__?.flushWaitQueue).exports.h;
let api = Object.values(wpRequire.c).find(x => x?.exports?.Bo?.get).exports.Bo;

const supportedTasks = ["WATCH_VIDEO", "PLAY_ON_DESKTOP", "STREAM_ON_DESKTOP", "PLAY_ACTIVITY", "WATCH_VIDEO_ON_MOBILE"]

let quests = [...QuestsStore.quests.values()].filter(
	x => x.userStatus?.enrolledAt &&
		!x.userStatus?.completedAt &&
		new Date(x.config.expiresAt).getTime() > Date.now() &&
		supportedTasks.find(y =>
			Object.keys((x.config.taskConfig ?? x.config.taskConfigV2).tasks).includes(y)
		)
)

let isApp = typeof DiscordNative !== "undefined"

if(quests.length === 0) {
	console.log("You don't have any uncompleted quests!")
} else {
	let doJob = function() {
		const quest = quests.pop()
		if(!quest) return

		const pid = Math.floor(Math.random() * 30000) + 1000
		
		const questName = quest.config.messages.questName
		const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2
		const taskName = supportedTasks.find(x => taskConfig.tasks[x] != null)
		const taskData = taskConfig.tasks[taskName]
		const applicationId = quest.config.application?.id ?? taskData.applications?.[0]?.id
		const secondsNeeded = taskData.target
		let secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0

		if(taskName === "WATCH_VIDEO" || taskName === "WATCH_VIDEO_ON_MOBILE") {
			const speed = 7
			let completed = false
			
			let fn = async () => {
				while(true) {
					const remaining = Math.min(speed, secondsNeeded - secondsDone)
					await new Promise(resolve => setTimeout(resolve, remaining * 1000))

					const timestamp = secondsDone + speed
					const res = await api.post({
						url: `/quests/${quest.id}/video-progress`,
						body: {
							timestamp: Math.min(secondsNeeded, timestamp + Math.random())
						}
					})

					completed = res.body.completed_at != null
					secondsDone = Math.min(secondsNeeded, timestamp)

					if(timestamp >= secondsNeeded) {
						break
					}
				}

				if(!completed) {
					await api.post({
						url: `/quests/${quest.id}/video-progress`,
						body: {
							timestamp: secondsNeeded
						}
					})
				}

				console.log("Quest completed!")
				doJob()
			}

			fn()
			console.log(`Spoofing video for ${questName}.`)
			
		} else if(taskName === "PLAY_ON_DESKTOP") {

			if(!isApp) {
				console.log(
					"This no longer works in browser for non-video quests. Use the discord desktop app to complete the",
					questName,
					"quest!"
				)
			} else {
				api.get({
					url: `/applications/public?application_ids=${applicationId}`
				}).then(res => {
					const appData = res.body[0]

					const exeName =
						appData.executables?.find(x => x.os === "win32")?.name?.replace(">","") ??
						appData.name.replace(/[\/\\:*?"<>|]/g, "")

					const fakeGame = {
						cmdLine: `C:\\Program Files\\${appData.name}\\${exeName}`,
						exeName,
						exePath: `c:/program files/${appData.name.toLowerCase()}/${exeName}`,
						hidden: false,
						isLauncher: false,
						id: applicationId,
						name: appData.name,
						pid: pid,
						pidPath: [pid],
						processName: appData.name,
						start: Date.now(),
					}

					const realGames = RunningGameStore.getRunningGames()
					const fakeGames = [fakeGame]

					const realGetRunningGames = RunningGameStore.getRunningGames
					const realGetGameForPID = RunningGameStore.getGameForPID

					RunningGameStore.getRunningGames = () => fakeGames
					RunningGameStore.getGameForPID = (pid) =>
						fakeGames.find(x => x.pid === pid)

					FluxDispatcher.dispatch({
						type: "RUNNING_GAMES_CHANGE",
						removed: realGames,
						added: [fakeGame],
						games: fakeGames
					})

					let fn = data => {
						let progress =
							quest.config.configVersion === 1
								? data.userStatus.streamProgressSeconds
								: Math.floor(data.userStatus.progress.PLAY_ON_DESKTOP.value)

						console.log(`Quest progress: ${progress}/${secondsNeeded}`)

						if(progress >= secondsNeeded) {
							console.log("Quest completed!")

							RunningGameStore.getRunningGames = realGetRunningGames
							RunningGameStore.getGameForPID = realGetGameForPID

							FluxDispatcher.dispatch({
								type: "RUNNING_GAMES_CHANGE",
								removed: [fakeGame],
								added: [],
								games: []
							})

							FluxDispatcher.unsubscribe(
								"QUESTS_SEND_HEARTBEAT_SUCCESS",
								fn
							)

							doJob()
						}
					}

					FluxDispatcher.subscribe(
						"QUESTS_SEND_HEARTBEAT_SUCCESS",
						fn
					)

					console.log(
						`Spoofed your game to ${appData.name}. Wait for ${Math.ceil(
							(secondsNeeded - secondsDone) / 60
						)} more minutes.`
					)
				})
			}

		} else if(taskName === "STREAM_ON_DESKTOP") {

			if(!isApp) {
				console.log(
					"This no longer works in browser for non-video quests. Use the discord desktop app to complete the",
					questName,
					"quest!"
				)
			} else {

				let realFunc =
					ApplicationStreamingStore.getStreamerActiveStreamMetadata

				ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({
					id: applicationId,
					pid,
					sourceName: null
				})

				let fn = data => {
					let progress =
						quest.config.configVersion === 1
							? data.userStatus.streamProgressSeconds
							: Math.floor(
								data.userStatus.progress.STREAM_ON_DESKTOP.value
							)

					console.log(
						`Quest progress: ${progress}/${secondsNeeded}`
					)

					if(progress >= secondsNeeded) {
						console.log("Quest completed!")

						ApplicationStreamingStore.getStreamerActiveStreamMetadata =
							realFunc

						FluxDispatcher.unsubscribe(
							"QUESTS_SEND_HEARTBEAT_SUCCESS",
							fn
						)

						doJob()
					}
				}

				FluxDispatcher.subscribe(
					"QUESTS_SEND_HEARTBEAT_SUCCESS",
					fn
				)

				console.log(
					`Spoofed your stream to the target game. Stream any window in vc for ${Math.ceil(
						(secondsNeeded - secondsDone) / 60
					)} more minutes.`
				)

				console.log(
					"Remember that you need at least 1 other person to be in the vc!"
				)
			}

		} else if(taskName === "PLAY_ACTIVITY") {

			const channelId =
				ChannelStore.getSortedPrivateChannels()[0]?.id ??
				Object.values(GuildChannelStore.getAllGuilds())
					.find(x => x != null && x.VOCAL.length > 0)
					.VOCAL[0].channel.id

			const streamKey = `call:${channelId}:1`
			
			let fn = async () => {
				console.log(
					"Completing quest",
					questName,
					"-",
					quest.config.messages.questName
				)

				while(true) {
					const res = await api.post({
						url: `/quests/${quest.id}/heartbeat`,
						body: {
							stream_key: streamKey,
							terminal: false
						}
					})

					const progress =
						res.body.progress.PLAY_ACTIVITY.value

					console.log(
						`Quest progress: ${progress}/${secondsNeeded}`
					)

					await new Promise(
						resolve => setTimeout(resolve, 20 * 1000)
					)

					if(progress >= secondsNeeded) {
						await api.post({
							url: `/quests/${quest.id}/heartbeat`,
							body: {
								stream_key: streamKey,
								terminal: true
							}
						})

						break
					}
				}

				console.log("Quest completed!")
				doJob()
			}

			fn()
		}
	}

	doJob()
}
```

> 💡 **Astuce :** si Discord bloque le collage dans la console, tapez d'abord `allow pasting`, puis appuyez sur **Entrée**. Vous pourrez ensuite coller le script.

---

## 🎮 5. Suivre les instructions affichées

Une fois le script lancé, regardez les messages affichés dans la console.

### 📺 Quête « Regarder une vidéo »

Si la quête demande de **regarder une vidéo**, vous pouvez simplement attendre.

### 🎮 Quête « Jouer à un jeu »

Si la quête demande de **jouer à un jeu**, vous pouvez simplement attendre que la progression atteigne l'objectif.

### 📡 Quête « Streamer un jeu »

Si la quête demande de **streamer un jeu** :

1. Rejoignez un salon vocal avec un ami ou un autre compte.
2. Lancez le partage d'écran.
3. Vous pouvez streamer n'importe quelle fenêtre.
4. Attendez que la progression soit terminée.

> ⚠️ Pour les quêtes de streaming, le script indique qu'au moins **une autre personne doit être présente dans le salon vocal**.

---

## ⏳ 6. Attendre la fin de la quête

Laissez le script fonctionner jusqu'à ce que la progression atteigne l'objectif.

Vous pouvez suivre l'avancement de deux manières :

* Dans la console, grâce aux messages :

  ```text
  Quest progress: X/Y
  ```
* Directement dans l'onglet **Quêtes** de Discord, grâce à la barre de progression.

Lorsque le script affiche :

```text
Quest completed!
```

la quête est terminée.

---

## 🎁 7. Récupérer la récompense

Une fois la quête terminée :

1. Retournez dans l'onglet **Quêtes**.
2. Ouvrez la quête terminée.
3. Cliquez sur **Réclamer la récompense**.

---

## 🛠️ Compatibilité

Le script prend actuellement en charge les types de quêtes suivants :

| Type                    | Description                    |
| ----------------------- | ------------------------------ |
| `WATCH_VIDEO`           | Regarder une vidéo             |
| `WATCH_VIDEO_ON_MOBILE` | Regarder une vidéo sur mobile  |
| `PLAY_ON_DESKTOP`       | Jouer à un jeu sur ordinateur  |
| `STREAM_ON_DESKTOP`     | Streamer un jeu sur ordinateur |
| `PLAY_ACTIVITY`         | Effectuer une activité         |

> ℹ️ Discord peut modifier régulièrement son fonctionnement interne. Si le script cesse de fonctionner, il peut être nécessaire de l'adapter à une nouvelle version de Discord.

---

## ❗ Dépannage

### Le script ne se colle pas dans la console

Essayez de taper :

```text
allow pasting
```

puis appuyez sur **Entrée** et réessayez.

### Le script indique qu'aucune quête n'est disponible

Si vous voyez :

```text
You don't have any uncompleted quests!
```

cela signifie qu'aucune quête compatible et non terminée n'a été détectée.

Vérifiez que :

* vous avez accepté la quête ;
* la quête n'est pas déjà terminée ;
* la quête n'a pas expiré ;
* le type de quête est pris en charge par le script.

### Une quête nécessite Discord Desktop

Certaines fonctionnalités ne fonctionnent plus depuis le navigateur.

Si le script affiche :

```text
This no longer works in browser for non-video quests.
```

utilisez l'application **Discord Desktop**.

---

## ⚠️ Avertissement

Ce projet n'est **pas affilié à Discord**.

L'utilisation de scripts qui modifient le comportement du client Discord peut présenter des risques et peut être contraire aux règles de la plateforme. Utilisez ce projet à vos propres risques.

Les développeurs de ce projet ne sont pas responsables des éventuelles conséquences liées à son utilisation.

---

## ⭐ Contribution

Les contributions, corrections et améliorations sont les bienvenues.

Si vous trouvez un bug ou si le script ne fonctionne plus après une mise à jour de Discord, vous pouvez ouvrir une **Issue** afin de signaler le problème.
