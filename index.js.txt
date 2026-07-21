import { Client, GatewayIntentBits, ChannelType, EmbedBuilder, ActionRowBuilder, ButtonBuilder, ButtonStyle, StringSelectMenuBuilder, ModalBuilder, TextInputBuilder, TextInputStyle, PermissionFlagsBits, Collection, SlashCommandBuilder, REST, Routes } from 'discord.js';

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMembers,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.DirectMessages,
    GatewayIntentBits.MessageContent,
  ],
});

const PREFIX = '.';
const tickets = new Collection();
const giveaways = new Collection();
const warns = new Collection();

const BUTTON_STYLE_MAP = {
  blue: ButtonStyle.Primary,
  gray: ButtonStyle.Secondary,
  grey: ButtonStyle.Secondary,
  green: ButtonStyle.Success,
  red: ButtonStyle.Danger,
};

// Returns true if the member is allowed to claim/manage tickets:
// either they have Manage Server, or they hold one of the configured support roles.
function isTicketStaff(member, guildTickets) {
  if (!member) return false;
  if (member.permissions.has(PermissionFlagsBits.ManageGuild)) return true;
  const roles = guildTickets?.roles || [];
  if (roles.length === 0) return false;
  return roles.some(roleId => member.roles.cache.has(roleId));
}

function getOrCreateGuildTickets(guildId) {
  if (!tickets.has(guildId)) {
    tickets.set(guildId, { roles: [], tickets: {}, panels: [] });
  }
  const guildTickets = tickets.get(guildId);
  if (!guildTickets.roles) guildTickets.roles = [];
  if (!guildTickets.tickets) guildTickets.tickets = {};
  if (!guildTickets.panels) guildTickets.panels = [];
  return guildTickets;
}


// Slash Commands
const commands = [
  new SlashCommandBuilder()
    .setName('ticketpanel')
    .setDescription('Create a ticket panel with up to 5 configurable buttons')
    .addStringOption(option => option.setName('title').setDescription('Panel title').setRequired(true))
    .addStringOption(option => option.setName('description').setDescription('Panel description').setRequired(true))
    .addStringOption(option => option.setName('color').setDescription('Panel color (hex code like #FF0000)').setRequired(false))
    .addStringOption(option => option.setName('button1_label').setDescription('Label for button 1 (default: Create Ticket)').setRequired(false))
    .addStringOption(option => option.setName('button1_emoji').setDescription('Emoji for button 1 (default: 🎫)').setRequired(false))
    .addStringOption(option => option.setName('button1_style').setDescription('Style for button 1').setRequired(false)
      .addChoices({ name: 'Blue', value: 'blue' }, { name: 'Gray', value: 'gray' }, { name: 'Green', value: 'green' }, { name: 'Red', value: 'red' }))
    .addStringOption(option => option.setName('button2_label').setDescription('Label for button 2 (adds a 2nd category)').setRequired(false))
    .addStringOption(option => option.setName('button2_emoji').setDescription('Emoji for button 2').setRequired(false))
    .addStringOption(option => option.setName('button2_style').setDescription('Style for button 2').setRequired(false)
      .addChoices({ name: 'Blue', value: 'blue' }, { name: 'Gray', value: 'gray' }, { name: 'Green', value: 'green' }, { name: 'Red', value: 'red' }))
    .addStringOption(option => option.setName('button3_label').setDescription('Label for button 3 (adds a 3rd category)').setRequired(false))
    .addStringOption(option => option.setName('button3_emoji').setDescription('Emoji for button 3').setRequired(false))
    .addStringOption(option => option.setName('button3_style').setDescription('Style for button 3').setRequired(false)
      .addChoices({ name: 'Blue', value: 'blue' }, { name: 'Gray', value: 'gray' }, { name: 'Green', value: 'green' }, { name: 'Red', value: 'red' }))
    .addStringOption(option => option.setName('button4_label').setDescription('Label for button 4 (adds a 4th category)').setRequired(false))
    .addStringOption(option => option.setName('button4_emoji').setDescription('Emoji for button 4').setRequired(false))
    .addStringOption(option => option.setName('button4_style').setDescription('Style for button 4').setRequired(false)
      .addChoices({ name: 'Blue', value: 'blue' }, { name: 'Gray', value: 'gray' }, { name: 'Green', value: 'green' }, { name: 'Red', value: 'red' }))
    .addStringOption(option => option.setName('button5_label').setDescription('Label for button 5 (adds a 5th category)').setRequired(false))
    .addStringOption(option => option.setName('button5_emoji').setDescription('Emoji for button 5').setRequired(false))
    .addStringOption(option => option.setName('button5_style').setDescription('Style for button 5').setRequired(false)
      .addChoices({ name: 'Blue', value: 'blue' }, { name: 'Gray', value: 'gray' }, { name: 'Green', value: 'green' }, { name: 'Red', value: 'red' }))
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild),
  
  new SlashCommandBuilder()
    .setName('ticketroles')
    .setDescription('Add or remove roles that can see tickets')
    .addRoleOption(option => option.setName('role').setDescription('Role to add/remove').setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild),
  
  new SlashCommandBuilder()
    .setName('giveaway')
    .setDescription('Start a giveaway with custom settings')
    .addStringOption(option => option.setName('prize').setDescription('Prize name').setRequired(true))
    .addIntegerOption(option => option.setName('duration').setDescription('Duration in minutes').setRequired(true))
    .addIntegerOption(option => option.setName('winners').setDescription('Number of winners').setRequired(true).setMinValue(1).setMaxValue(100))
    .addStringOption(option => option.setName('description').setDescription('Giveaway description').setRequired(false))
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild),
  
  new SlashCommandBuilder()
    .setName('endgiveaway')
    .setDescription('End a giveaway immediately and pick winners')
    .addStringOption(option => option.setName('giveaway_id').setDescription('Giveaway ID').setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild),
  
  new SlashCommandBuilder()
    .setName('rerollgiveaway')
    .setDescription('Reroll giveaway winners')
    .addStringOption(option => option.setName('giveaway_id').setDescription('Giveaway ID').setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild),
  
  new SlashCommandBuilder()
    .setName('giveawaystatus')
    .setDescription('Check giveaway status')
    .addStringOption(option => option.setName('giveaway_id').setDescription('Giveaway ID').setRequired(true)),
  
  new SlashCommandBuilder()
    .setName('mute')
    .setDescription('Mute a user')
    .addUserOption(option => option.setName('user').setDescription('User to mute').setRequired(true))
    .addIntegerOption(option => option.setName('minutes').setDescription('Duration in minutes').setRequired(false))
    .addStringOption(option => option.setName('reason').setDescription('Reason for mute').setRequired(false))
    .setDefaultMemberPermissions(PermissionFlagsBits.ModerateMembers),
  
  new SlashCommandBuilder()
    .setName('unmute')
    .setDescription('Unmute a user')
    .addUserOption(option => option.setName('user').setDescription('User to unmute').setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.ModerateMembers),
  
  new SlashCommandBuilder()
    .setName('ban')
    .setDescription('Ban a user')
    .addUserOption(option => option.setName('user').setDescription('User to ban').setRequired(true))
    .addStringOption(option => option.setName('reason').setDescription('Reason for ban').setRequired(false))
    .setDefaultMemberPermissions(PermissionFlagsBits.BanMembers),
  
  new SlashCommandBuilder()
    .setName('unban')
    .setDescription('Unban a user')
    .addStringOption(option => option.setName('user_id').setDescription('User ID to unban').setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.BanMembers),
  
  new SlashCommandBuilder()
    .setName('kick')
    .setDescription('Kick a user')
    .addUserOption(option => option.setName('user').setDescription('User to kick').setRequired(true))
    .addStringOption(option => option.setName('reason').setDescription('Reason for kick').setRequired(false))
    .setDefaultMemberPermissions(PermissionFlagsBits.KickMembers),
  
  new SlashCommandBuilder()
    .setName('warn')
    .setDescription('Warn a user')
    .addUserOption(option => option.setName('user').setDescription('User to warn').setRequired(true))
    .addStringOption(option => option.setName('reason').setDescription('Reason for warning').setRequired(false))
    .setDefaultMemberPermissions(PermissionFlagsBits.ModerateMembers),
  
  new SlashCommandBuilder()
    .setName('warnings')
    .setDescription('Check warnings for a user')
    .addUserOption(option => option.setName('user').setDescription('User to check').setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.ModerateMembers),
  
  new SlashCommandBuilder()
    .setName('clearwarnings')
    .setDescription('Clear warnings for a user')
    .addUserOption(option => option.setName('user').setDescription('User to clear warnings').setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild),
  
  new SlashCommandBuilder()
    .setName('slowmode')
    .setDescription('Set slowmode for a channel')
    .addIntegerOption(option => option.setName('seconds').setDescription('Slowmode duration in seconds').setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageChannels),
  
  new SlashCommandBuilder()
    .setName('lock')
    .setDescription('Lock a channel')
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageChannels),
  
  new SlashCommandBuilder()
    .setName('unlock')
    .setDescription('Unlock a channel')
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageChannels),
  
  new SlashCommandBuilder()
    .setName('purge')
    .setDescription('Delete messages from a channel')
    .addIntegerOption(option => option.setName('amount').setDescription('Number of messages to delete').setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageMessages),
  
  new SlashCommandBuilder()
    .setName('userinfo')
    .setDescription('Get information about a user')
    .addUserOption(option => option.setName('user').setDescription('User to check').setRequired(false)),
  
  new SlashCommandBuilder()
    .setName('serverinfo')
    .setDescription('Get information about the server'),
  
  new SlashCommandBuilder()
    .setName('claim')
    .setDescription('Claim a ticket (support staff only)'),

  new SlashCommandBuilder()
    .setName('unclaim')
    .setDescription('Release your claim on a ticket'),
  
  new SlashCommandBuilder()
    .setName('closeticket')
    .setDescription('Close the current ticket')
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild),
  
  new SlashCommandBuilder()
    .setName('tickettranscript')
    .setDescription('Get transcript of current ticket'),
  
  new SlashCommandBuilder()
    .setName('ticketstats')
    .setDescription('View ticket statistics')
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild),
  
  new SlashCommandBuilder()
    .setName('addticketuser')
    .setDescription('Add a user to the ticket')
    .addUserOption(option => option.setName('user').setDescription('User to add').setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild),
  
  new SlashCommandBuilder()
    .setName('removeticketuser')
    .setDescription('Remove a user from the ticket')
    .addUserOption(option => option.setName('user').setDescription('User to remove').setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild),
  
  new SlashCommandBuilder()
    .setName('renameticket')
    .setDescription('Rename the ticket')
    .addStringOption(option => option.setName('name').setDescription('New ticket name').setRequired(true))
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild),
];

client.once('ready', async () => {
  console.log(`✅ Bot logged in as ${client.user.tag}`);
  client.user.setActivity('tickets & giveaways', { type: 'WATCHING' });

  try {
    const rest = new REST({ version: '10' }).setToken(process.env.DISCORD_TOKEN);
    await rest.put(Routes.applicationCommands(client.user.id), { body: commands });
    console.log('✅ Slash commands registered');
  } catch (error) {
    console.error('Error registering commands:', error);
  }
});

client.on('messageCreate', async (message) => {
  if (!message.content.startsWith(PREFIX) || message.author.bot) return;

  const args = message.content.slice(PREFIX.length).trim().split(/ +/);
  const command = args.shift().toLowerCase();

  try {
    if (command === 'help') {
      const embed = new EmbedBuilder()
        .setColor('#0099ff')
        .setTitle('📖 Bot Commands')
        .setDescription('Use `/` for slash commands or `.` for prefix commands')
        .addFields(
          { name: '🎫 Tickets (Manage Guild)', value: '`/ticketpanel` • `/ticketroles` • `/ticketstats` • `/closeticket` • `/addticketuser` • `/removeticketuser` • `/renameticket` • `/tickettranscript` • `/claim` • `/unclaim` (claim/unclaim are support-staff only)' },
          { name: '🎉 Giveaways (Manage Guild)', value: '`/giveaway` • `/endgiveaway` • `/rerollgiveaway` • `/giveawaystatus`' },
          { name: '🔨 Moderation', value: '`/mute` • `/unmute` • `/ban` • `/unban` • `/kick` • `/warn` • `/warnings` • `/clearwarnings`' },
          { name: '⚙️ Utilities', value: '`/slowmode` • `/lock` • `/unlock` • `/purge` • `/userinfo` • `/serverinfo`' }
        );

      message.reply({ embeds: [embed] });
    }

    if (command === 'claim') {
      const channel = message.channel;
      if (!channel.name.startsWith('ticket-')) {
        return message.reply('❌ This is not a ticket channel');
      }

      const guildTickets = getOrCreateGuildTickets(message.guildId);

      if (!isTicketStaff(message.member, guildTickets)) {
        return message.reply('❌ Only support staff can claim tickets. Ask a server admin to add your role with `/ticketroles`.');
      }

      const ticketData = guildTickets.tickets[channel.id];
      if (ticketData && ticketData.claimed) {
        return message.reply(`❌ Already claimed by <@${ticketData.claimedBy}>`);
      }

      if (!ticketData) {
        guildTickets.tickets[channel.id] = { claimedBy: message.author.id, claimed: true, createdAt: Date.now(), users: [message.author.id] };
      } else {
        ticketData.claimedBy = message.author.id;
        ticketData.claimed = true;
      }

      const embed = new EmbedBuilder()
        .setColor('#00ff00')
        .setDescription(`✅ Ticket claimed by ${message.author}`);

      message.reply({ embeds: [embed] });
    }

    if (command === 'closeticket') {
      if (!message.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
        return message.reply('❌ You need **Manage Server** permission');
      }

      const channel = message.channel;
      if (!channel.name.startsWith('ticket-')) {
        return message.reply('❌ This is not a ticket channel');
      }

      const embed = new EmbedBuilder()
        .setColor('#ff0000')
        .setDescription('🔒 Ticket closed');

      await message.reply({ embeds: [embed] });
      setTimeout(() => channel.delete(), 2000);
    }

    if (command === 'transcript') {
      const channel = message.channel;
      if (!channel.name.startsWith('ticket-')) {
        return message.reply('❌ This is not a ticket channel');
      }

      const messages = await channel.messages.fetch({ limit: 100 });
      const transcript = messages
        .reverse()
        .map(m => `[${new Date(m.createdTimestamp).toLocaleString()}] ${m.author.tag}: ${m.content}`)
        .join('\n');

      const embed = new EmbedBuilder()
        .setColor('#0099ff')
        .setTitle(`📋 Ticket Transcript - ${channel.name}`)
        .setDescription(`\`\`\`\n${transcript || 'No messages'}\n\`\`\``)
        .setFooter({ text: `Requested by ${message.author.tag}` });

      message.reply({ embeds: [embed] });
    }

    if (command === 'adduser') {
      if (!message.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
        return message.reply('❌ You need **Manage Server** permission');
      }

      const channel = message.channel;
      if (!channel.name.startsWith('ticket-')) {
        return message.reply('❌ This is not a ticket channel');
      }

      const user = message.mentions.users.first();
      if (!user) return message.reply('❌ Mention a user');

      await channel.permissionOverwrites.create(user.id, {
        ViewChannel: true,
        SendMessages: true,
      });

      message.reply(`✅ Added ${user} to the ticket`);
    }

    if (command === 'removeuser') {
      if (!message.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
        return message.reply('❌ You need **Manage Server** permission');
      }

      const channel = message.channel;
      if (!channel.name.startsWith('ticket-')) {
        return message.reply('❌ This is not a ticket channel');
      }

      const user = message.mentions.users.first();
      if (!user) return message.reply('❌ Mention a user');

      await channel.permissionOverwrites.delete(user.id);

      message.reply(`✅ Removed ${user} from the ticket`);
    }

    if (command === 'renameticket') {
      if (!message.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
        return message.reply('❌ You need **Manage Server** permission');
      }

      const channel = message.channel;
      if (!channel.name.startsWith('ticket-')) {
        return message.reply('❌ This is not a ticket channel');
      }

      const newName = args.join(' ');
      if (!newName) return message.reply('❌ Provide a new name');

      await channel.setName(`ticket-${newName}`);

      message.reply(`✅ Ticket renamed to \`ticket-${newName}\``);
    }

    if (command === 'mute') {
      if (!message.member.permissions.has(PermissionFlagsBits.ModerateMembers)) {
        return message.reply('❌ You need **Moderate Members** permission');
      }

      const user = message.mentions.users.first();
      if (!user) return message.reply('❌ Mention a user');

      const minutes = parseInt(args[1]) || 60;
      const reason = args.slice(2).join(' ') || 'No reason provided';

      const member = await message.guild.members.fetch(user.id);
      await member.timeout(minutes * 60000, reason);

      const embed = new EmbedBuilder()
        .setColor('#ff9900')
        .setTitle('🔇 User Muted')
        .setDescription(`**User:** ${user}\n**Duration:** ${minutes} minutes\n**Reason:** ${reason}`);

      message.reply({ embeds: [embed] });
    }

    if (command === 'unmute') {
      if (!message.member.permissions.has(PermissionFlagsBits.ModerateMembers)) {
        return message.reply('❌ You need **Moderate Members** permission');
      }

      const user = message.mentions.users.first();
      if (!user) return message.reply('❌ Mention a user');

      const member = await message.guild.members.fetch(user.id);
      await member.timeout(null);

      const embed = new EmbedBuilder()
        .setColor('#00ff00')
        .setTitle('🔊 User Unmuted')
        .setDescription(`**User:** ${user}`);

      message.reply({ embeds: [embed] });
    }

    if (command === 'ban') {
      if (!message.member.permissions.has(PermissionFlagsBits.BanMembers)) {
        return message.reply('❌ You need **Ban Members** permission');
      }

      const user = message.mentions.users.first();
      if (!user) return message.reply('❌ Mention a user');

      const reason = args.slice(1).join(' ') || 'No reason provided';

      await message.guild.members.ban(user, { reason });

      const embed = new EmbedBuilder()
        .setColor('#ff0000')
        .setTitle('🔨 User Banned')
        .setDescription(`**User:** ${user}\n**Reason:** ${reason}`);

      message.reply({ embeds: [embed] });
    }

    if (command === 'unban') {
      if (!message.member.permissions.has(PermissionFlagsBits.BanMembers)) {
        return message.reply('❌ You need **Ban Members** permission');
      }

      const userId = args[0];
      if (!userId) return message.reply('❌ Provide a user ID');

      await message.guild.bans.remove(userId);

      message.reply(`✅ User ${userId} unbanned`);
    }

    if (command === 'kick') {
      if (!message.member.permissions.has(PermissionFlagsBits.KickMembers)) {
        return message.reply('❌ You need **Kick Members** permission');
      }

      const user = message.mentions.users.first();
      if (!user) return message.reply('❌ Mention a user');

      const reason = args.slice(1).join(' ') || 'No reason provided';

      const member = await message.guild.members.fetch(user.id);
      await member.kick(reason);

      const embed = new EmbedBuilder()
        .setColor('#ff6600')
        .setTitle('👢 User Kicked')
        .setDescription(`**User:** ${user}\n**Reason:** ${reason}`);

      message.reply({ embeds: [embed] });
    }

    if (command === 'warn') {
      if (!message.member.permissions.has(PermissionFlagsBits.ModerateMembers)) {
        return message.reply('❌ You need **Moderate Members** permission');
      }

      const user = message.mentions.users.first();
      if (!user) return message.reply('❌ Mention a user');

      const reason = args.slice(1).join(' ') || 'No reason';

      if (!warns.has(message.guildId)) {
        warns.set(message.guildId, {});
      }

      const guildWarns = warns.get(message.guildId);
      if (!guildWarns[user.id]) {
        guildWarns[user.id] = [];
      }

      guildWarns[user.id].push({
        reason,
        moderator: message.author.tag,
        timestamp: Date.now(),
      });

      const embed = new EmbedBuilder()
        .setColor('#ff9900')
        .setTitle('⚠️ User Warned')
        .setDescription(`**User:** ${user}\n**Reason:** ${reason}\n**Total Warnings:** ${guildWarns[user.id].length}`);

      message.reply({ embeds: [embed] });
    }

    if (command === 'warnings') {
      if (!message.member.permissions.has(PermissionFlagsBits.ModerateMembers)) {
        return message.reply('❌ You need **Moderate Members** permission');
      }

      const user = message.mentions.users.first();
      if (!user) return message.reply('❌ Mention a user');

      if (!warns.has(message.guildId) || !warns.get(message.guildId)[user.id]) {
        return message.reply(`✅ ${user} has no warnings`);
      }

      const userWarns = warns.get(message.guildId)[user.id];
      const warnList = userWarns
        .map((w, i) => `**${i + 1}.** ${w.reason} (by ${w.moderator})`)
        .join('\n');

      const embed = new EmbedBuilder()
        .setColor('#ff9900')
        .setTitle(`⚠️ Warnings for ${user.tag}`)
        .setDescription(warnList)
        .setFooter({ text: `Total: ${userWarns.length}` });

      message.reply({ embeds: [embed] });
    }

    if (command === 'clearwarnings') {
      if (!message.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
        return message.reply('❌ You need **Manage Server** permission');
      }

      const user = message.mentions.users.first();
      if (!user) return message.reply('❌ Mention a user');

      if (warns.has(message.guildId)) {
        delete warns.get(message.guildId)[user.id];
      }

      message.reply(`✅ Cleared all warnings for ${user}`);
    }

    if (command === 'slowmode') {
      if (!message.member.permissions.has(PermissionFlagsBits.ManageChannels)) {
        return message.reply('❌ You need **Manage Channels** permission');
      }

      const seconds = parseInt(args[0]);
      if (!seconds) return message.reply('❌ Provide seconds');

      await message.channel.setRateLimitPerUser(seconds);

      message.reply(`✅ Slowmode set to ${seconds} seconds`);
    }

    if (command === 'lock') {
      if (!message.member.permissions.has(PermissionFlagsBits.ManageChannels)) {
        return message.reply('❌ You need **Manage Channels** permission');
      }

      await message.channel.permissionOverwrites.edit(message.guildId, {
        SendMessages: false,
      });

      message.reply('🔒 Channel locked');
    }

    if (command === 'unlock') {
      if (!message.member.permissions.has(PermissionFlagsBits.ManageChannels)) {
        return message.reply('❌ You need **Manage Channels** permission');
      }

      await message.channel.permissionOverwrites.edit(message.guildId, {
        SendMessages: null,
      });

      message.reply('🔓 Channel unlocked');
    }

    if (command === 'purge') {
      if (!message.member.permissions.has(PermissionFlagsBits.ManageMessages)) {
        return message.reply('❌ You need **Manage Messages** permission');
      }

      const amount = parseInt(args[0]);
      if (!amount || amount > 100) return message.reply('❌ Provide amount (max 100)');

      await message.channel.bulkDelete(amount);
      message.reply(`✅ Deleted ${amount} messages`);
    }

    if (command === 'userinfo') {
      const user = message.mentions.users.first() || message.author;
      const member = await message.guild.members.fetch(user.id);

      const embed = new EmbedBuilder()
        .setColor('#0099ff')
        .setTitle(`👤 ${user.tag}`)
        .setThumbnail(user.displayAvatarURL())
        .addFields(
          { name: 'User ID', value: user.id, inline: true },
          { name: 'Account Created', value: `<t:${Math.floor(user.createdTimestamp / 1000)}:R>`, inline: true },
          { name: 'Joined Server', value: `<t:${Math.floor(member.joinedTimestamp / 1000)}:R>`, inline: true },
          { name: 'Roles', value: member.roles.cache.map(r => r).join(', ') || 'None', inline: false }
        );

      message.reply({ embeds: [embed] });
    }

    if (command === 'serverinfo') {
      const guild = message.guild;

      const embed = new EmbedBuilder()
        .setColor('#0099ff')
        .setTitle(`🏢 ${guild.name}`)
        .setThumbnail(guild.iconURL())
        .addFields(
          { name: 'Server ID', value: guild.id, inline: true },
          { name: 'Members', value: `${guild.memberCount}`, inline: true },
          { name: 'Channels', value: `${guild.channels.cache.size}`, inline: true },
          { name: 'Roles', value: `${guild.roles.cache.size}`, inline: true },
          { name: 'Owner', value: `<@${guild.ownerId}>`, inline: true },
          { name: 'Created', value: `<t:${Math.floor(guild.createdTimestamp / 1000)}:R>`, inline: true }
        );

      message.reply({ embeds: [embed] });
    }
  } catch (error) {
    console.error('Error in message command:', error);
    message.reply('❌ An error occurred').catch(() => {});
  }
});

client.on('interactionCreate', async (interaction) => {
  if (interaction.isCommand()) {
    const { commandName } = interaction;

    try {
      // TICKET PANEL COMMAND
      if (commandName === 'ticketpanel') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
          return interaction.reply({ content: '❌ You need **Manage Server** permission', ephemeral: true });
        }

        const title = interaction.options.getString('title');
        const description = interaction.options.getString('description');
        const color = interaction.options.getString('color') || '#0099ff';

        const guildTickets = getOrCreateGuildTickets(interaction.guildId);
        const panelId = `panel_${Date.now()}`;

        // Gather up to 5 buttons. Button 1 falls back to sensible defaults so a bare
        // /ticketpanel still works; buttons 2-5 only appear if a label was given for them.
        const buttonDefs = [];
        for (let i = 1; i <= 5; i++) {
          const label = interaction.options.getString(`button${i}_label`);
          if (i > 1 && !label) continue; // only button 1 is guaranteed to exist

          const emoji = interaction.options.getString(`button${i}_emoji`) || (i === 1 ? '🎫' : '📩');
          const styleKey = interaction.options.getString(`button${i}_style`) || (i === 1 ? 'blue' : 'gray');
          const style = BUTTON_STYLE_MAP[styleKey] || ButtonStyle.Primary;

          buttonDefs.push({
            index: i,
            label: label || 'Create Ticket',
            emoji,
            style: styleKey,
          });
        }

        const embed = new EmbedBuilder()
          .setColor(color)
          .setTitle(title)
          .setDescription(description)
          .addFields(
            { name: '📋 How to use', value: buttonDefs.length > 1
              ? 'Click a button below to open a ticket for that category'
              : 'Click the button below to create a support ticket', inline: false }
          )
          .setFooter({ text: 'Ticket System' });

        // Discord allows a max of 5 buttons per row, which matches our max of 5 categories.
        const row = new ActionRowBuilder().addComponents(
          buttonDefs.map(b => new ButtonBuilder()
            .setCustomId(`create_ticket_${panelId}_${b.index}`)
            .setLabel(b.label)
            .setStyle(BUTTON_STYLE_MAP[b.style] || ButtonStyle.Primary)
            .setEmoji(b.emoji))
        );

        const msg = await interaction.channel.send({ embeds: [embed], components: [row] });

        guildTickets.panels.push({
          id: panelId,
          messageId: msg.id,
          channelId: interaction.channelId,
          title,
          description,
          color,
          buttons: buttonDefs,
        });

        const buttonSummary = buttonDefs.map(b => `${b.emoji} ${b.label}`).join('\n');
        interaction.reply({ content: `✅ **Ticket panel created!**\n\n📌 Panel ID: \`${panelId}\`\n📝 Title: ${title}\n🎨 Color: ${color}\n\n**Buttons (${buttonDefs.length}):**\n${buttonSummary}`, ephemeral: true });
      }

      if (commandName === 'ticketroles') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
          return interaction.reply({ content: '❌ You need **Manage Server** permission', ephemeral: true });
        }

        const role = interaction.options.getRole('role');
        const guildTickets = getOrCreateGuildTickets(interaction.guildId);

        if (guildTickets.roles.includes(role.id)) {
          guildTickets.roles = guildTickets.roles.filter(r => r !== role.id);
          interaction.reply({ content: `✅ Removed ${role} from ticket access`, ephemeral: true });
        } else {
          guildTickets.roles.push(role.id);
          interaction.reply({ content: `✅ Added ${role} to ticket access`, ephemeral: true });
        }
      }

      if (commandName === 'claim') {
        const channel = interaction.channel;
        if (!channel.name.startsWith('ticket-')) {
          return interaction.reply({ content: '❌ This is not a ticket channel', ephemeral: true });
        }

        const guildTickets = getOrCreateGuildTickets(interaction.guildId);

        if (!isTicketStaff(interaction.member, guildTickets)) {
          return interaction.reply({ content: '❌ Only support staff can claim tickets. Ask a server admin to add your role with `/ticketroles`.', ephemeral: true });
        }

        const ticketData = guildTickets.tickets[channel.id];
        if (ticketData && ticketData.claimed) {
          return interaction.reply({ content: `❌ Already claimed by <@${ticketData.claimedBy}>`, ephemeral: true });
        }

        if (!ticketData) {
          guildTickets.tickets[channel.id] = { claimedBy: interaction.user.id, claimed: true, createdAt: Date.now(), users: [interaction.user.id] };
        } else {
          ticketData.claimedBy = interaction.user.id;
          ticketData.claimed = true;
        }

        const embed = new EmbedBuilder()
          .setColor('#00ff00')
          .setDescription(`✅ Ticket claimed by ${interaction.user}`);

        interaction.reply({ embeds: [embed] });
      }

      if (commandName === 'unclaim') {
        const channel = interaction.channel;
        if (!channel.name.startsWith('ticket-')) {
          return interaction.reply({ content: '❌ This is not a ticket channel', ephemeral: true });
        }

        const guildTickets = getOrCreateGuildTickets(interaction.guildId);
        const ticketData = guildTickets.tickets[channel.id];

        if (!ticketData || !ticketData.claimed) {
          return interaction.reply({ content: '❌ This ticket is not claimed', ephemeral: true });
        }

        const isClaimer = ticketData.claimedBy === interaction.user.id;
        const isManager = interaction.member.permissions.has(PermissionFlagsBits.ManageGuild);
        if (!isClaimer && !isManager) {
          return interaction.reply({ content: `❌ Only <@${ticketData.claimedBy}> or a manager can unclaim this ticket`, ephemeral: true });
        }

        ticketData.claimed = false;
        ticketData.claimedBy = null;

        interaction.reply({ embeds: [new EmbedBuilder().setColor('#ff9900').setDescription('🔓 Ticket unclaimed — anyone from support can claim it now')] });
      }

      if (commandName === 'closeticket') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
          return interaction.reply({ content: '❌ You need **Manage Server** permission', ephemeral: true });
        }

        const channel = interaction.channel;
        if (!channel.name.startsWith('ticket-')) {
          return interaction.reply({ content: '❌ This is not a ticket channel', ephemeral: true });
        }

        const embed = new EmbedBuilder()
          .setColor('#ff0000')
          .setDescription('🔒 Ticket closed');

        await interaction.reply({ embeds: [embed] });
        setTimeout(() => channel.delete(), 2000);
      }

      if (commandName === 'tickettranscript') {
        const channel = interaction.channel;
        if (!channel.name.startsWith('ticket-')) {
          return interaction.reply({ content: '❌ This is not a ticket channel', ephemeral: true });
        }

        await interaction.deferReply();

        const messages = await channel.messages.fetch({ limit: 100 });
        const transcript = messages
          .reverse()
          .map(m => `[${new Date(m.createdTimestamp).toLocaleString()}] ${m.author.tag}: ${m.content}`)
          .join('\n');

        const embed = new EmbedBuilder()
          .setColor('#0099ff')
          .setTitle(`📋 Ticket Transcript - ${channel.name}`)
          .setDescription(`\`\`\`\n${transcript || 'No messages'}\n\`\`\``)
          .setFooter({ text: `Requested by ${interaction.user.tag}` });

        interaction.editReply({ embeds: [embed] });
      }

      if (commandName === 'ticketstats') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
          return interaction.reply({ content: '❌ You need **Manage Server** permission', ephemeral: true });
        }

        if (!tickets.has(interaction.guildId)) {
          return interaction.reply({ content: '❌ No tickets yet', ephemeral: true });
        }

        const guildTickets = tickets.get(interaction.guildId);
        const totalTickets = Object.keys(guildTickets.tickets || {}).length;
        const claimedTickets = Object.values(guildTickets.tickets || {}).filter(t => t.claimed).length;

        const embed = new EmbedBuilder()
          .setColor('#0099ff')
          .setTitle('📊 Ticket Statistics')
          .addFields(
            { name: 'Total Tickets', value: `${totalTickets}`, inline: true },
            { name: 'Claimed Tickets', value: `${claimedTickets}`, inline: true },
            { name: 'Unclaimed Tickets', value: `${totalTickets - claimedTickets}`, inline: true }
          );

        interaction.reply({ embeds: [embed] });
      }

      if (commandName === 'addticketuser') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
          return interaction.reply({ content: '❌ You need **Manage Server** permission', ephemeral: true });
        }

        const channel = interaction.channel;
        if (!channel.name.startsWith('ticket-')) {
          return interaction.reply({ content: '❌ This is not a ticket channel', ephemeral: true });
        }

        const user = interaction.options.getUser('user');
        await channel.permissionOverwrites.create(user.id, {
          ViewChannel: true,
          SendMessages: true,
        });

        interaction.reply({ content: `✅ Added ${user} to the ticket`, ephemeral: true });
      }

      if (commandName === 'removeticketuser') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
          return interaction.reply({ content: '❌ You need **Manage Server** permission', ephemeral: true });
        }

        const channel = interaction.channel;
        if (!channel.name.startsWith('ticket-')) {
          return interaction.reply({ content: '❌ This is not a ticket channel', ephemeral: true });
        }

        const user = interaction.options.getUser('user');
        await channel.permissionOverwrites.delete(user.id);

        interaction.reply({ content: `✅ Removed ${user} from the ticket`, ephemeral: true });
      }

      if (commandName === 'renameticket') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
          return interaction.reply({ content: '❌ You need **Manage Server** permission', ephemeral: true });
        }

        const channel = interaction.channel;
        if (!channel.name.startsWith('ticket-')) {
          return interaction.reply({ content: '❌ This is not a ticket channel', ephemeral: true });
        }

        const newName = interaction.options.getString('name');
        await channel.setName(`ticket-${newName}`);

        interaction.reply({ content: `✅ Ticket renamed to \`ticket-${newName}\``, ephemeral: true });
      }

      // GIVEAWAY COMMANDS
      if (commandName === 'giveaway') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
          return interaction.reply({ content: '❌ You need **Manage Server** permission', ephemeral: true });
        }

        const prize = interaction.options.getString('prize');
        const duration = interaction.options.getInteger('duration');
        const winners = interaction.options.getInteger('winners');
        const description = interaction.options.getString('description') || 'Click to participate!';

        const giveawayId = `giveaway_${Date.now()}`;
        const endTime = Date.now() + duration * 60000;

        giveaways.set(giveawayId, {
          prize,
          participants: new Set(),
          active: true,
          createdAt: Date.now(),
          endTime,
          duration,
          winners,
          description,
          messageId: null,
          channelId: interaction.channelId,
          guildId: interaction.guildId,
          createdBy: interaction.user.id,
        });

        const timeRemaining = `<t:${Math.floor(endTime / 1000)}:R>`;

        const embed = new EmbedBuilder()
          .setColor('#FFD700')
          .setTitle('🎉 GIVEAWAY')
          .setDescription(description)
          .addFields(
            { name: '🎁 Prize', value: prize, inline: true },
            { name: '👥 Winners', value: `${winners}`, inline: true },
            { name: '⏰ Ends', value: timeRemaining, inline: true },
            { name: '📊 Participants', value: '0', inline: true }
          )
          .setFooter({ text: `Giveaway ID: ${giveawayId}` });

        const row = new ActionRowBuilder().addComponents(
          new ButtonBuilder()
            .setCustomId(`join_giveaway_${giveawayId}`)
            .setLabel('Join Giveaway')
            .setStyle(ButtonStyle.Success)
            .setEmoji('🎁'),
          new ButtonBuilder()
            .setCustomId(`end_giveaway_${giveawayId}`)
            .setLabel('End Now')
            .setStyle(ButtonStyle.Danger)
            .setEmoji('⏹️')
        );

        const giveawayMsg = await interaction.channel.send({ embeds: [embed], components: [row] });
        giveaways.get(giveawayId).messageId = giveawayMsg.id;

        interaction.reply({ content: `✅ Giveaway started!\n\n**ID:** \`${giveawayId}\`\n**Prize:** ${prize}\n**Winners:** ${winners}\n**Duration:** ${duration} minutes`, ephemeral: true });

        // Auto-end giveaway
        setTimeout(async () => {
          const giveaway = giveaways.get(giveawayId);
          if (giveaway && giveaway.active) {
            giveaway.active = false;
            
            if (giveaway.participants.size === 0) {
              const noWinnersEmbed = new EmbedBuilder()
                .setColor('#FF0000')
                .setTitle('🎉 GIVEAWAY ENDED')
                .setDescription(`**Prize:** ${giveaway.prize}\n**Result:** No participants!`);
              
              try {
                const channel = await client.channels.fetch(giveaway.channelId);
                const msg = await channel.messages.fetch(giveaway.messageId);
                await msg.edit({ embeds: [noWinnersEmbed], components: [] });
              } catch (e) {}
            } else {
              const winnersList = [];
              const participantsArray = Array.from(giveaway.participants);
              
              for (let i = 0; i < Math.min(giveaway.winners, participantsArray.length); i++) {
                const randomIndex = Math.floor(Math.random() * participantsArray.length);
                winnersList.push(participantsArray[randomIndex]);
                participantsArray.splice(randomIndex, 1);
              }

              const winnersText = winnersList.map(id => `<@${id}>`).join(', ');

              const endEmbed = new EmbedBuilder()
                .setColor('#FFD700')
                .setTitle('🎉 GIVEAWAY ENDED')
                .addFields(
                  { name: '🎁 Prize', value: giveaway.prize, inline: true },
                  { name: '👥 Winners', value: `${giveaway.winners}`, inline: true },
                  { name: '📊 Participants', value: `${giveaway.participants.size}`, inline: true },
                  { name: '🏆 Winner(s)', value: winnersText, inline: false }
                );

              try {
                const channel = await client.channels.fetch(giveaway.channelId);
                const msg = await channel.messages.fetch(giveaway.messageId);
                await msg.edit({ embeds: [endEmbed], components: [] });
              } catch (e) {}
            }
          }
        }, duration * 60000);
      }

      if (commandName === 'endgiveaway') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
          return interaction.reply({ content: '❌ You need **Manage Server** permission', ephemeral: true });
        }

        const giveawayId = interaction.options.getString('giveaway_id');
        if (!giveawayId || !giveaways.has(giveawayId)) {
          return interaction.reply({ content: '❌ Giveaway not found', ephemeral: true });
        }

        const giveaway = giveaways.get(giveawayId);
        giveaway.active = false;

        if (giveaway.participants.size === 0) {
          const noWinnersEmbed = new EmbedBuilder()
            .setColor('#FF0000')
            .setTitle('🎉 GIVEAWAY ENDED')
            .setDescription(`**Prize:** ${giveaway.prize}\n**Result:** No participants!`);

          interaction.reply({ embeds: [noWinnersEmbed] });
          
          try {
            const channel = await client.channels.fetch(giveaway.channelId);
            const msg = await channel.messages.fetch(giveaway.messageId);
            await msg.edit({ embeds: [noWinnersEmbed], components: [] });
          } catch (e) {}
          return;
        }

        const winnersList = [];
        const participantsArray = Array.from(giveaway.participants);
        
        for (let i = 0; i < Math.min(giveaway.winners, participantsArray.length); i++) {
          const randomIndex = Math.floor(Math.random() * participantsArray.length);
          winnersList.push(participantsArray[randomIndex]);
          participantsArray.splice(randomIndex, 1);
        }

        const winnersText = winnersList.map(id => `<@${id}>`).join(', ');

        const embed = new EmbedBuilder()
          .setColor('#FFD700')
          .setTitle('🎉 GIVEAWAY ENDED')
          .addFields(
            { name: '🎁 Prize', value: giveaway.prize, inline: true },
            { name: '👥 Winners', value: `${giveaway.winners}`, inline: true },
            { name: '📊 Participants', value: `${giveaway.participants.size}`, inline: true },
            { name: '🏆 Winner(s)', value: winnersText, inline: false }
          );

        interaction.reply({ embeds: [embed] });

        try {
          const channel = await client.channels.fetch(giveaway.channelId);
          const msg = await channel.messages.fetch(giveaway.messageId);
          await msg.edit({ embeds: [embed], components: [] });
        } catch (e) {}
      }

      if (commandName === 'rerollgiveaway') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
          return interaction.reply({ content: '❌ You need **Manage Server** permission', ephemeral: true });
        }

        const giveawayId = interaction.options.getString('giveaway_id');
        if (!giveawayId || !giveaways.has(giveawayId)) {
          return interaction.reply({ content: '❌ Giveaway not found', ephemeral: true });
        }

        const giveaway = giveaways.get(giveawayId);
        if (giveaway.participants.size === 0) {
          return interaction.reply({ content: '❌ No participants to reroll', ephemeral: true });
        }

        const winnersList = [];
        const participantsArray = Array.from(giveaway.participants);
        
        for (let i = 0; i < Math.min(giveaway.winners, participantsArray.length); i++) {
          const randomIndex = Math.floor(Math.random() * participantsArray.length);
          winnersList.push(participantsArray[randomIndex]);
          participantsArray.splice(randomIndex, 1);
        }

        const winnersText = winnersList.map(id => `<@${id}>`).join(', ');

        const embed = new EmbedBuilder()
          .setColor('#FFD700')
          .setTitle('🎉 GIVEAWAY REROLLED')
          .addFields(
            { name: '🎁 Prize', value: giveaway.prize, inline: true },
            { name: '👥 Winners', value: `${giveaway.winners}`, inline: true },
            { name: '🏆 New Winner(s)', value: winnersText, inline: false }
          );

        interaction.reply({ embeds: [embed] });
      }

      if (commandName === 'giveawaystatus') {
        const giveawayId = interaction.options.getString('giveaway_id');
        if (!giveawayId || !giveaways.has(giveawayId)) {
          return interaction.reply({ content: '❌ Giveaway not found', ephemeral: true });
        }

        const giveaway = giveaways.get(giveawayId);
        const timeRemaining = giveaway.active ? `<t:${Math.floor(giveaway.endTime / 1000)}:R>` : 'Ended';
        const status = giveaway.active ? '🟢 Active' : '🔴 Ended';

        const embed = new EmbedBuilder()
          .setColor('#0099ff')
          .setTitle('📊 Giveaway Status')
          .addFields(
            { name: 'ID', value: giveawayId, inline: true },
            { name: 'Status', value: status, inline: true },
            { name: '🎁 Prize', value: giveaway.prize, inline: true },
            { name: '👥 Winners', value: `${giveaway.winners}`, inline: true },
            { name: '📊 Participants', value: `${giveaway.participants.size}`, inline: true },
            { name: '⏰ Ends', value: timeRemaining, inline: true }
          );

        interaction.reply({ embeds: [embed] });
      }

      // MODERATION COMMANDS
      if (commandName === 'mute') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ModerateMembers)) {
          return interaction.reply({ content: '❌ You need **Moderate Members** permission', ephemeral: true });
        }

        const user = interaction.options.getUser('user');
        const minutes = interaction.options.getInteger('minutes') || 60;
        const reason = interaction.options.getString('reason') || 'No reason provided';

        const member = await interaction.guild.members.fetch(user.id);
        await member.timeout(minutes * 60000, reason);

        const embed = new EmbedBuilder()
          .setColor('#ff9900')
          .setTitle('🔇 User Muted')
          .setDescription(`**User:** ${user}\n**Duration:** ${minutes} minutes\n**Reason:** ${reason}`);

        interaction.reply({ embeds: [embed] });
      }

      if (commandName === 'unmute') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ModerateMembers)) {
          return interaction.reply({ content: '❌ You need **Moderate Members** permission', ephemeral: true });
        }

        const user = interaction.options.getUser('user');
        const member = await interaction.guild.members.fetch(user.id);
        await member.timeout(null);

        const embed = new EmbedBuilder()
          .setColor('#00ff00')
          .setTitle('🔊 User Unmuted')
          .setDescription(`**User:** ${user}`);

        interaction.reply({ embeds: [embed] });
      }

      if (commandName === 'ban') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.BanMembers)) {
          return interaction.reply({ content: '❌ You need **Ban Members** permission', ephemeral: true });
        }

        const user = interaction.options.getUser('user');
        const reason = interaction.options.getString('reason') || 'No reason provided';

        await interaction.guild.members.ban(user, { reason });

        const embed = new EmbedBuilder()
          .setColor('#ff0000')
          .setTitle('🔨 User Banned')
          .setDescription(`**User:** ${user}\n**Reason:** ${reason}`);

        interaction.reply({ embeds: [embed] });
      }

      if (commandName === 'unban') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.BanMembers)) {
          return interaction.reply({ content: '❌ You need **Ban Members** permission', ephemeral: true });
        }

        const userId = interaction.options.getString('user_id');
        await interaction.guild.bans.remove(userId);

        interaction.reply({ content: `✅ User ${userId} unbanned`, ephemeral: true });
      }

      if (commandName === 'kick') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.KickMembers)) {
          return interaction.reply({ content: '❌ You need **Kick Members** permission', ephemeral: true });
        }

        const user = interaction.options.getUser('user');
        const reason = interaction.options.getString('reason') || 'No reason provided';

        const member = await interaction.guild.members.fetch(user.id);
        await member.kick(reason);

        const embed = new EmbedBuilder()
          .setColor('#ff6600')
          .setTitle('👢 User Kicked')
          .setDescription(`**User:** ${user}\n**Reason:** ${reason}`);

        interaction.reply({ embeds: [embed] });
      }

      if (commandName === 'warn') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ModerateMembers)) {
          return interaction.reply({ content: '❌ You need **Moderate Members** permission', ephemeral: true });
        }

        const user = interaction.options.getUser('user');
        const reason = interaction.options.getString('reason') || 'No reason';

        if (!warns.has(interaction.guildId)) {
          warns.set(interaction.guildId, {});
        }

        const guildWarns = warns.get(interaction.guildId);
        if (!guildWarns[user.id]) {
          guildWarns[user.id] = [];
        }

        guildWarns[user.id].push({
          reason,
          moderator: interaction.user.tag,
          timestamp: Date.now(),
        });

        const embed = new EmbedBuilder()
          .setColor('#ff9900')
          .setTitle('⚠️ User Warned')
          .setDescription(`**User:** ${user}\n**Reason:** ${reason}\n**Total Warnings:** ${guildWarns[user.id].length}`);

        interaction.reply({ embeds: [embed] });
      }

      if (commandName === 'warnings') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ModerateMembers)) {
          return interaction.reply({ content: '❌ You need **Moderate Members** permission', ephemeral: true });
        }

        const user = interaction.options.getUser('user');

        if (!warns.has(interaction.guildId) || !warns.get(interaction.guildId)[user.id]) {
          return interaction.reply({ content: `✅ ${user} has no warnings`, ephemeral: true });
        }

        const userWarns = warns.get(interaction.guildId)[user.id];
        const warnList = userWarns
          .map((w, i) => `**${i + 1}.** ${w.reason} (by ${w.moderator})`)
          .join('\n');

        const embed = new EmbedBuilder()
          .setColor('#ff9900')
          .setTitle(`⚠️ Warnings for ${user.tag}`)
          .setDescription(warnList)
          .setFooter({ text: `Total: ${userWarns.length}` });

        interaction.reply({ embeds: [embed] });
      }

      if (commandName === 'clearwarnings') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
          return interaction.reply({ content: '❌ You need **Manage Server** permission', ephemeral: true });
        }

        const user = interaction.options.getUser('user');

        if (warns.has(interaction.guildId)) {
          delete warns.get(interaction.guildId)[user.id];
        }

        interaction.reply({ content: `✅ Cleared all warnings for ${user}`, ephemeral: true });
      }

      // UTILITY COMMANDS
      if (commandName === 'slowmode') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageChannels)) {
          return interaction.reply({ content: '❌ You need **Manage Channels** permission', ephemeral: true });
        }

        const seconds = interaction.options.getInteger('seconds');
        await interaction.channel.setRateLimitPerUser(seconds);

        interaction.reply({ content: `✅ Slowmode set to ${seconds} seconds`, ephemeral: true });
      }

      if (commandName === 'lock') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageChannels)) {
          return interaction.reply({ content: '❌ You need **Manage Channels** permission', ephemeral: true });
        }

        await interaction.channel.permissionOverwrites.edit(interaction.guildId, {
          SendMessages: false,
        });

        interaction.reply({ content: '🔒 Channel locked', ephemeral: true });
      }

      if (commandName === 'unlock') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageChannels)) {
          return interaction.reply({ content: '❌ You need **Manage Channels** permission', ephemeral: true });
        }

        await interaction.channel.permissionOverwrites.edit(interaction.guildId, {
          SendMessages: null,
        });

        interaction.reply({ content: '🔓 Channel unlocked', ephemeral: true });
      }

      if (commandName === 'purge') {
        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageMessages)) {
          return interaction.reply({ content: '❌ You need **Manage Messages** permission', ephemeral: true });
        }

        const amount = interaction.options.getInteger('amount');
        if (amount > 100) {
          return interaction.reply({ content: '❌ Max 100 messages', ephemeral: true });
        }

        await interaction.channel.bulkDelete(amount);
        interaction.reply({ content: `✅ Deleted ${amount} messages`, ephemeral: true });
      }

      if (commandName === 'userinfo') {
        const user = interaction.options.getUser('user') || interaction.user;
        const member = await interaction.guild.members.fetch(user.id);

        const embed = new EmbedBuilder()
          .setColor('#0099ff')
          .setTitle(`👤 ${user.tag}`)
          .setThumbnail(user.displayAvatarURL())
          .addFields(
            { name: 'User ID', value: user.id, inline: true },
            { name: 'Account Created', value: `<t:${Math.floor(user.createdTimestamp / 1000)}:R>`, inline: true },
            { name: 'Joined Server', value: `<t:${Math.floor(member.joinedTimestamp / 1000)}:R>`, inline: true },
            { name: 'Roles', value: member.roles.cache.map(r => r).join(', ') || 'None', inline: false }
          );

        interaction.reply({ embeds: [embed] });
      }

      if (commandName === 'serverinfo') {
        const guild = interaction.guild;

        const embed = new EmbedBuilder()
          .setColor('#0099ff')
          .setTitle(`🏢 ${guild.name}`)
          .setThumbnail(guild.iconURL())
          .addFields(
            { name: 'Server ID', value: guild.id, inline: true },
            { name: 'Members', value: `${guild.memberCount}`, inline: true },
            { name: 'Channels', value: `${guild.channels.cache.size}`, inline: true },
            { name: 'Roles', value: `${guild.roles.cache.size}`, inline: true },
            { name: 'Owner', value: `<@${guild.ownerId}>`, inline: true },
            { name: 'Created', value: `<t:${Math.floor(guild.createdTimestamp / 1000)}:R>`, inline: true }
          );

        interaction.reply({ embeds: [embed] });
      }
    } catch (error) {
      console.error('Error in slash command:', error);
      interaction.reply({ content: '❌ An error occurred', ephemeral: true }).catch(() => {});
    }
  }

  // BUTTON INTERACTIONS
  if (interaction.isButton()) {
    try {
      if (interaction.customId.startsWith('create_ticket_')) {
        // customId shape: create_ticket_<panelId>_<buttonIndex>
        const raw = interaction.customId.replace('create_ticket_', '');
        const lastUnderscore = raw.lastIndexOf('_');
        const panelId = raw.slice(0, lastUnderscore);
        const buttonIndex = parseInt(raw.slice(lastUnderscore + 1), 10) || 1;

        const guildTickets = getOrCreateGuildTickets(interaction.guildId);
        const panel = guildTickets.panels?.find(p => p.id === panelId);

        if (!panel) {
          return interaction.reply({ content: '❌ Panel not found. Please create a new panel with `/ticketpanel`', ephemeral: true });
        }

        const buttonDef = panel.buttons?.find(b => b.index === buttonIndex) || panel.buttons?.[0];
        const categoryLabel = buttonDef?.label || panel.title;

        // Prevent a user from piling up multiple open tickets on this panel.
        const existingTicketEntry = Object.entries(guildTickets.tickets).find(
          ([, t]) => t.creator === interaction.user.id && !t.closed && t.panelId === panelId
        );
        if (existingTicketEntry) {
          const [existingChannelId] = existingTicketEntry;
          return interaction.reply({ content: `❌ You already have an open ticket: <#${existingChannelId}>`, ephemeral: true });
        }

        await interaction.deferReply({ ephemeral: true });

        const ticketNumber = Math.floor(Math.random() * 10000);
        const channel = await interaction.guild.channels.create({
          name: `ticket-${interaction.user.username}-${ticketNumber}`,
          type: ChannelType.GuildText,
          permissionOverwrites: [
            {
              id: interaction.guild.id,
              deny: [PermissionFlagsBits.ViewChannel],
            },
            {
              id: interaction.user.id,
              allow: [PermissionFlagsBits.ViewChannel, PermissionFlagsBits.SendMessages],
            },
          ],
        });

        guildTickets.tickets[channel.id] = {
          creator: interaction.user.id,
          claimed: false,
          claimedBy: null,
          createdAt: Date.now(),
          users: [interaction.user.id],
          category: categoryLabel,
          panelId,
          closed: false,
        };

        if (guildTickets.roles && guildTickets.roles.length > 0) {
          for (const roleId of guildTickets.roles) {
            await channel.permissionOverwrites.create(roleId, {
              ViewChannel: true,
              SendMessages: true,
            }).catch(() => {});
          }
        }

        const embed = new EmbedBuilder()
          .setColor(panel.color || '#0099ff')
          .setTitle(`🎫 ${categoryLabel}`)
          .setDescription(`**Ticket created by:** ${interaction.user}\n**Category:** ${categoryLabel}\n\n${panel.description}`)
          .addFields({ name: 'Status', value: '🟡 Unclaimed', inline: true })
          .setFooter({ text: `Ticket #${ticketNumber}` })
          .setTimestamp();

        const row = new ActionRowBuilder().addComponents(
          new ButtonBuilder()
            .setCustomId('claim_ticket')
            .setLabel('Claim Ticket')
            .setStyle(ButtonStyle.Primary)
            .setEmoji('✋'),
          new ButtonBuilder()
            .setCustomId('close_ticket')
            .setLabel('Close Ticket')
            .setStyle(ButtonStyle.Danger)
            .setEmoji('🔒')
        );

        const mentionRoles = (guildTickets.roles || []).map(r => `<@&${r}>`).join(' ');
        await channel.send({ content: `${interaction.user}${mentionRoles ? ' ' + mentionRoles : ''}`, embeds: [embed], components: [row] });
        interaction.editReply({ content: `✅ Ticket created: ${channel}` });
      }

      if (interaction.customId === 'close_ticket') {
        const channel = interaction.channel;
        const guildTickets = getOrCreateGuildTickets(interaction.guildId);
        const ticketData = guildTickets.tickets[channel.id];
        if (ticketData) ticketData.closed = true;

        const embed = new EmbedBuilder()
          .setColor('#ff0000')
          .setDescription(`🔒 Ticket closed by ${interaction.user}. Deleting in a few seconds...`);

        await interaction.reply({ embeds: [embed] });
        setTimeout(() => channel.delete().catch(() => {}), 5000);
      }

      if (interaction.customId === 'claim_ticket' || interaction.customId === 'unclaim_ticket') {
        const channel = interaction.channel;
        const guildTickets = getOrCreateGuildTickets(interaction.guildId);
        const ticketData = guildTickets.tickets[channel.id] || (guildTickets.tickets[channel.id] = {
          creator: null, claimed: false, claimedBy: null, createdAt: Date.now(), users: [], closed: false,
        });

        const claiming = interaction.customId === 'claim_ticket';

        if (claiming) {
          if (!isTicketStaff(interaction.member, guildTickets)) {
            return interaction.reply({ content: '❌ Only support staff can claim tickets.', ephemeral: true });
          }
          if (ticketData.claimed) {
            return interaction.reply({ content: `❌ Already claimed by <@${ticketData.claimedBy}>`, ephemeral: true });
          }
          ticketData.claimed = true;
          ticketData.claimedBy = interaction.user.id;
        } else {
          const isClaimer = ticketData.claimedBy === interaction.user.id;
          const isManager = interaction.member.permissions.has(PermissionFlagsBits.ManageGuild);
          if (!ticketData.claimed) {
            return interaction.reply({ content: '❌ This ticket is not claimed', ephemeral: true });
          }
          if (!isClaimer && !isManager) {
            return interaction.reply({ content: `❌ Only <@${ticketData.claimedBy}> or a manager can unclaim this ticket`, ephemeral: true });
          }
          ticketData.claimed = false;
          ticketData.claimedBy = null;
        }

        // Rebuild the buttons: swap Claim <-> Unclaim, keep Close always available.
        const newRow = new ActionRowBuilder().addComponents(
          ticketData.claimed
            ? new ButtonBuilder().setCustomId('unclaim_ticket').setLabel('Unclaim').setStyle(ButtonStyle.Secondary).setEmoji('🔓')
            : new ButtonBuilder().setCustomId('claim_ticket').setLabel('Claim Ticket').setStyle(ButtonStyle.Primary).setEmoji('✋'),
          new ButtonBuilder().setCustomId('close_ticket').setLabel('Close Ticket').setStyle(ButtonStyle.Danger).setEmoji('🔒')
        );

        // Update the status field on the original ticket embed if we can find it.
        const original = interaction.message;
        let newEmbeds = original.embeds;
        if (original.embeds?.[0]) {
          const rebuilt = EmbedBuilder.from(original.embeds[0]);
          const fields = (original.embeds[0].fields || []).filter(f => f.name !== 'Status');
          fields.push({
            name: 'Status',
            value: ticketData.claimed ? `🟢 Claimed by <@${ticketData.claimedBy}>` : '🟡 Unclaimed',
            inline: true,
          });
          rebuilt.setFields(fields);
          newEmbeds = [rebuilt];
        }

        await interaction.update({ embeds: newEmbeds, components: [newRow] });
        await interaction.followUp({
          embeds: [new EmbedBuilder()
            .setColor(ticketData.claimed ? '#00ff00' : '#ff9900')
            .setDescription(ticketData.claimed ? `✅ Ticket claimed by ${interaction.user}` : `🔓 Ticket unclaimed by ${interaction.user}`)],
        });
      }

      if (interaction.customId.startsWith('join_giveaway_')) {
        const giveawayId = interaction.customId.replace('join_giveaway_', '');
        const giveaway = giveaways.get(giveawayId);

        if (!giveaway || !giveaway.active) {
          return interaction.reply({ content: '❌ Giveaway ended', ephemeral: true });
        }

        if (giveaway.participants.has(interaction.user.id)) {
          giveaway.participants.delete(interaction.user.id);
          interaction.reply({ content: `❌ Removed from giveaway (${giveaway.participants.size} participants)`, ephemeral: true });
        } else {
          giveaway.participants.add(interaction.user.id);
          interaction.reply({ content: `✅ Joined! (${giveaway.participants.size} participants)`, ephemeral: true });
        }
      }

      if (interaction.customId.startsWith('end_giveaway_')) {
        const giveawayId = interaction.customId.replace('end_giveaway_', '');
        const giveaway = giveaways.get(giveawayId);

        if (!giveaway) {
          return interaction.reply({ content: '❌ Giveaway not found', ephemeral: true });
        }

        if (!interaction.member.permissions.has(PermissionFlagsBits.ManageGuild)) {
          return interaction.reply({ content: '❌ You need **Manage Server** permission', ephemeral: true });
        }

        giveaway.active = false;

        if (giveaway.participants.size === 0) {
          const noWinnersEmbed = new EmbedBuilder()
            .setColor('#FF0000')
            .setTitle('🎉 GIVEAWAY ENDED')
            .setDescription(`**Prize:** ${giveaway.prize}\n**Result:** No participants!`);

          interaction.reply({ embeds: [noWinnersEmbed] });
          
          try {
            const msg = await interaction.channel.messages.fetch(giveaway.messageId);
            await msg.edit({ embeds: [noWinnersEmbed], components: [] });
          } catch (e) {}
          return;
        }

        const winnersList = [];
        const participantsArray = Array.from(giveaway.participants);
        
        for (let i = 0; i < Math.min(giveaway.winners, participantsArray.length); i++) {
          const randomIndex = Math.floor(Math.random() * participantsArray.length);
          winnersList.push(participantsArray[randomIndex]);
          participantsArray.splice(randomIndex, 1);
        }

        const winnersText = winnersList.map(id => `<@${id}>`).join(', ');

        const embed = new EmbedBuilder()
          .setColor('#FFD700')
          .setTitle('🎉 GIVEAWAY ENDED')
          .addFields(
            { name: '🎁 Prize', value: giveaway.prize, inline: true },
            { name: '👥 Winners', value: `${giveaway.winners}`, inline: true },
            { name: '📊 Participants', value: `${giveaway.participants.size}`, inline: true },
            { name: '🏆 Winner(s)', value: winnersText, inline: false }
          );

        interaction.reply({ embeds: [embed] });

        try {
          const msg = await interaction.channel.messages.fetch(giveaway.messageId);
          await msg.edit({ embeds: [embed], components: [] });
        } catch (e) {}
      }
    } catch (error) {
      console.error('Error in button interaction:', error);
      interaction.reply({ content: '❌ An error occurred', ephemeral: true }).catch(() => {});
    }
  }
});

client.login(process.env.DISCORD_TOKEN);
