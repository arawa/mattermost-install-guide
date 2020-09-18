# Mattermost Arawa specific configuration

* This is the config we have been using at our corporate Mattermost instance.

## Specific Mattermost configuration

* Enabling the ability to [upload plugins](https://docs.mattermost.com/administration/plugins.html#custom-plugins) via the system console by changing the value of the key `EnableUploads` to `true` in the file `/etc/mattermost/config.json`
* `System Console` > `Integrations` > `GIF (Beta)` > `Enable GIF Picker`: `true`
* `System Console` > `Site Configuration` > `Posts` > `Enable Link Previews`: `true`
* `System Console` > `Environment` > `SMTP` > configure all the needed fields

## Plugins installed

* GitHub
* Jitsi Meet

## Plugins to consider

* [Autolink](https://github.com/mattermost/mattermost-plugin-autolink#configuration-management)
* [Collabora Mattermost](https://www.collaboraoffice.com/integrations/mattermost-plugin/)
* [Big Thumbnails](https://github.com/wget/mattermost-plugin-big-thumbnails)
