import discord
from discord.ext import commands, tasks
import random
import youtube_dl
import asyncio

bot = commands.Bot(command_prefix="|", description = "Music Bot !,préfixe | .")
bot.remove_command('help')
status = ["|help",
           "T'ambiancé !"] 
           
musics = {}
ytdl = youtube_dl.YoutubeDL()


@bot.event
async def on_ready():
    print("Ready")
    changeStatus.start()

@tasks.loop(seconds = 5)
async def changeStatus():
    game = discord.Game(random.choice(status))
    await bot.change_presence(status = discord.Status.dnd, activity = game)

class Video:
    def __init__(self, link):
        video = ytdl.extract_info(link, download=False)
        video_format = video["formats"][0]
        self.url = video["webpage_url"]
        self.stream_url = video_format["url"]

@bot.command()
async def leave(ctx):
    client = ctx.guild.voice_client
    await client.disconnect()
    musics[ctx.guild] = []

@bot.command()
async def resume(ctx):
    client = ctx.guild.voice_client
    if client.is_paused():
        client.resume()


@bot.command()
async def pause(ctx):
    client = ctx.guild.voice_client
    if not client.is_paused():
        client.pause()


@bot.command()
async def skip(ctx):
    client = ctx.guild.voice_client
    client.stop()


def play_song(client, queue, song):
    source = discord.PCMVolumeTransformer(discord.FFmpegPCMAudio(song.stream_url
        , before_options = "-reconnect 1 -reconnect_streamed 1 -reconnect_delay_max 5"))

    def next(_):
        if len(queue) > 0:
            new_song = queue[0]
            del queue[0]
            play_song(client, queue, new_song)
        else:
            asyncio.run_coroutine_threadsafe(client.disconnect(), bot.loop)

    client.play(source, after=next)


@bot.command()
async def play(ctx, url):
    print("play")
    client = ctx.guild.voice_client

    if client and client.channel:
        video = Video(url)
        musics[ctx.guild].append(video)
    else:
        channel = ctx.author.voice.channel
        video = Video(url)
        musics[ctx.guild] = []
        client = await channel.connect()
        await ctx.send(f"Je lance : {video.url}")
        play_song(client, musics[ctx.guild], video)

        

@bot.command(pass_context=True)
async def help(ctx):
    author = ctx.message.author

    test_e = discord.Embed(
        colour=discord.Colour.orange()
    )
    test_e.set_author(name="Bot prefix = |")
    test_e.add_field(name="play (url)", value="play this music in your vocal", inline=False)
    test_e.add_field(name="pause", value="pause")
    test_e.add_field(name="resume", value="resume", inline=False)
    test_e.add_field(name="skip", value="skip next", inline=False)
    test_e.add_field(name="leave", value="make him leave", inline=False)

    await author.send(embed=test_e)


