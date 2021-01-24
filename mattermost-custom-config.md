# Mattermost Arawa specific configuration

* This is the config we have been using at our corporate Mattermost instance.

## Specific Mattermost configuration

* Enabling the ability to [upload plugins](https://docs.mattermost.com/administration/plugins.html#custom-plugins) via the system console by changing the value of the key `EnableUploads` to `true` in the file `/etc/mattermost/config.json`
* `System Console` > `Integrations` > `GIF (Beta)` > `Enable GIF Picker`: `true`
* `System Console` > `Site Configuration` > `Posts` > `Enable Link Previews`: `true`
* `System Console` > `Site Configuration` > `Posts` > `Enable Latex Rendering`: `true`
* `System Console` > `Site Configuration` > `Users and Teams` > `Teammate Name Display`: `Show first and last name`.
* `System Console` > `Environment` > `SMTP` > configure all the needed fields (ideally use a "do not reply" email address).
* `System Console` > `Site Configuration` > `Notifications` > `Notification Display Name`: `Arawa Chat`
* `System Console` > `Site Configuration` > `Notifications` > `Notification From Address`, specify the "do not reply" address you defined in the SMTP section above, otherwize the From header field in the email will be empty meaning the email won't be authenticated and the DKIM signature verification will fail.

* If your users haven't specified their first and last name you'll have to specify them manually. Use a simple SQL script to do it, and match on the username you know.

  ```
  MariaDB [(none)]> use mattermostdb;
  MariaDB [mattermostdb]> show tables;
  +------------------------+
  | Tables_in_mattermostdb |
  +------------------------+
  | Audits                 |
  | Bots                   |
  | ChannelMemberHistory   |
  | ChannelMembers         |
  | Channels               |
  | ClusterDiscovery       |
  | CommandWebhooks        |
  | Commands               |
  | Compliances            |
  | Emoji                  |
  | FileInfo               |
  | GroupChannels          |
  | GroupMembers           |
  | GroupTeams             |
  | IncomingWebhooks       |
  | Jobs                   |
  | Licenses               |
  | LinkMetadata           |
  | OAuthAccessData        |
  | OAuthApps              |
  | OAuthAuthData          |
  | OutgoingWebhooks       |
  | PluginKeyValueStore    |
  | Posts                  |
  | Preferences            |
  | PublicChannels         |
  | Reactions              |
  | Roles                  |
  | Schemes                |
  | Sessions               |
  | SidebarCategories      |
  | SidebarChannels        |
  | Status                 |
  | Systems                |
  | TeamMembers            |
  | Teams                  |
  | TermsOfService         |
  | Tokens                 |
  | UserAccessTokens       |
  | UserGroups             |
  | UserTermsOfService     |
  | Users                  |
  +------------------------+
  42 rows in set (0.001 sec)
  MariaDB [mattermostdb]> describe Users;
  +--------------------+--------------+------+-----+---------+-------+
  | Field              | Type         | Null | Key | Default | Extra |
  +--------------------+--------------+------+-----+---------+-------+
  | Id                 | varchar(26)  | NO   | PRI | NULL    |       |
  | CreateAt           | bigint(20)   | YES  | MUL | NULL    |       |
  | UpdateAt           | bigint(20)   | YES  | MUL | NULL    |       |
  | DeleteAt           | bigint(20)   | YES  | MUL | NULL    |       |
  | Username           | varchar(64)  | YES  | UNI | NULL    |       |
  | Password           | varchar(128) | YES  |     | NULL    |       |
  | AuthData           | varchar(128) | YES  | UNI | NULL    |       |
  | AuthService        | varchar(32)  | YES  |     | NULL    |       |
  | Email              | varchar(128) | YES  | UNI | NULL    |       |
  | EmailVerified      | tinyint(1)   | YES  |     | NULL    |       |
  | Nickname           | varchar(64)  | YES  |     | NULL    |       |
  | FirstName          | varchar(64)  | YES  |     | NULL    |       |
  | LastName           | varchar(64)  | YES  |     | NULL    |       |
  | Position           | varchar(128) | YES  |     | NULL    |       |
  | Roles              | text         | YES  |     | NULL    |       |
  | AllowMarketing     | tinyint(1)   | YES  |     | NULL    |       |
  | Props              | text         | YES  |     | NULL    |       |
  | NotifyProps        | text         | YES  |     | NULL    |       |
  | LastPasswordUpdate | bigint(20)   | YES  |     | NULL    |       |
  | LastPictureUpdate  | bigint(20)   | YES  |     | NULL    |       |
  | FailedAttempts     | int(11)      | YES  |     | NULL    |       |
  | Locale             | varchar(5)   | YES  |     | NULL    |       |
  | Timezone           | text         | YES  |     | NULL    |       |
  | MfaActive          | tinyint(1)   | YES  |     | NULL    |       |
  | MfaSecret          | varchar(128) | YES  |     | NULL    |       |
  +--------------------+--------------+------+-----+---------+-------+
  25 rows in set (0.002 sec)
  MariaDB [mattermostdb]> update Users set FirstName = 'William', LastName = 'Gathoye' where UserName = 'wget';

## Plugins installed

* GitHub
* Jitsi Meet

## Plugins to consider

* [Autolink](https://github.com/mattermost/mattermost-plugin-autolink#configuration-management)
* [Collabora Mattermost](https://www.collaboraoffice.com/integrations/mattermost-plugin/)
* [Big Thumbnails](https://github.com/wget/mattermost-plugin-big-thumbnails)
