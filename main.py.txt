import discord
from discord.ext import commands
import yt_dlp
import asyncio
import re

intents = discord.Intents.default()
intents.message_content = True
intents.voice_states = True

bot = commands.Bot(command_prefix="!", intents=intents)

queues = {}

ydl_opts = {
    'format': 'bestaudio/best',
    'noplaylist': False,
    'quiet': True,
    'no_warnings': True,
    'default_search': 'ytsearch',
}

ffmpeg_options = {
    'before_options': '-reconnect 1 -reconnect_streamed 1 -reconnect_delay_max 5',
    'options': '-vn',
}

@bot.event
async def on_ready():
    print(f"البوت شغال باسم: {bot.user}")

async def play_next(ctx):
    guild_id = ctx.guild.id
    if guild_id not in queues or not queues[guild_id]:
        return

    voice = ctx.voice_client
    if not voice:
        return

    url, title = queues[guild_id].pop(0)
    source = discord.FFmpegPCMAudio(url, **ffmpeg_options)
    voice.play(source, after=lambda e: asyncio.run_coroutine_threadsafe(play_next(ctx), bot.loop))
    await ctx.send(f"**{title}** الآن يشغل")

@bot.event
async def on_message(message):
    if message.author.bot:
        return

    content = message.content.strip()

    # توكل = يطلع
    if content == "توكل":
        if message.guild.voice_client:
            await message.guild.voice_client.disconnect()
            await message.channel.send("طلعت من الروم")
        else:
            await message.channel.send("أنا مو في روم")
        return

    # غير = سكيب
    if content == "غير":
        if message.guild.voice_client and message.guild.voice_client.is_playing():
            message.guild.voice_client.stop()
            await message.channel.send("تم التغيير")
        else:
            await message.channel.send("ما في شيء يشتغل")
        return

    # p = تشغيل
    if content.startswith("p ") or content.startswith("p"):
        query = content[1:].strip()
        
        # يصلح الرابط لو نقص حرف
        if "ttp" in query:
            query = query.replace("ttp", "http", 1)
        if "youtube" in query and not query.startswith("http"):
            query = "https://" + query.lstrip(":/")

        if not message.author.voice:
            await message.channel.send("ادخل روم صوتي أولاً")
            return

        if not message.guild.voice_client:
            await message.author.voice.channel.connect()

        voice = message.guild.voice_client
        guild_id = message.guild.id

        if guild_id not in queues:
            queues[guild_id] = []

        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            try:
                info = ydl.extract_info(query, download=False)

                if 'entries' in info and info['entries']:
                    entries = [e for e in info['entries'] if e]
                    for entry in entries:
                        url = entry['url']
                        title = entry.get('title', 'أغنية')
                        queues[guild_id].append((url, title))
                    await message.channel.send(f"تمت إضافة {len(entries)} أغنية من البلاي لست")
                else:
                    url = info['url']
                    title = info.get('title', 'أغنية')
                    queues[guild_id].append((url, title))
                    await message.channel.send(f"تمت إضافة **{title}**")
            except Exception as e:
                await message.channel.send(f"خطأ: {e}")
                return

        ctx = await bot.get_context(message)
        if not voice.is_playing() and not voice.is_paused():
            await play_next(ctx)
        return

    await bot.process_commands(message)

# حط التوكن هنا
bot.run(os.environ.get("DISCORD_TOKEN"))
