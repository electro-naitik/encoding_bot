# encoding_bot
this is encoding bot which can add hardsubtitle to video which is shared to him 
import os
import logging
import tempfile

START_TIME = time.time()

# Admin configuration
ADMIN_IDS = [6629249301]
BANNED_USERS = set()
USER_STATS = {}

class DownloadProgress:
    def __init__(self):
        self.start_time = time.time()
        self.downloaded = 0
        self.total_size = 0
        self.last_update = 0
        
    def format_size(self, size_bytes):
        """Convert bytes to human readable format"""
        if size_bytes == 0:
            return "0B"
        size_names = ["B", "KB", "MB", "GB", "TB"]
        i = int(math.floor(math.log(size_bytes, 1024)))
        p = math.pow(1024, i)
        s = round(size_bytes / p, 2)
        return f"{s} {size_names[i]}"
    
    def format_speed(self, bytes_per_sec):
        """Convert speed to human readable format"""
        return self.format_size(bytes_per_sec) + "/s"
    
    def format_time(self, seconds):
        """Format time to MM:SS or HH:MM:SS"""
        if seconds < 60:
            return f"{int(seconds)}s"
        elif seconds < 3600:
            minutes = int(seconds // 60)
            seconds = int(seconds % 60)
            return f"{minutes}m{seconds}s"
        else:
            hours = int(seconds // 3600)
            minutes = int((seconds % 3600) // 60)
            return f"{hours}h{minutes}m"
    
    def get_progress_bar(self, percentage, length=20):
        """Create a progress bar"""
        filled = int(length * percentage / 100)
        empty = length - filled
        bar_chars = ["█", "▓", "▒", "░"]
        return bar_chars[0] * filled + bar_chars[3] * empty
    
    def update(self, downloaded, total_size):
        """Update progress values"""
        self.downloaded = downloaded
        self.total_size = total_size
        self.last_update = time.time()

class FileUploader:
    """Handle file uploads to various services for public links"""
    
    @staticmethod
    async def upload_to_fileio(file_path, filename):
        """Upload file to file.io service"""
        try:
            with open(file_path, 'rb') as file:
                files = {'file': (filename, file)}
                response = requests.post(
                    'https://file.io', 
                    files=files, 
                    timeout=30,
                    headers={'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'}
                )
                data = response.json()
                
            if data.get('success'):
                return {
                    'success': True,
                    'url': data['link'],
                    'expires': data.get('expires', '14 days'),
                    'service': 'file.io 🌐',
                    'emoji': '📎'
                }
            else:
                return {'success': False, 'error': data.get('message', 'Upload failed')}
                
        except Exception as e:
            return {'success': False, 'error': str(e)}
    
    @staticmethod
    async def upload_to_transfersh(file_path, filename):
        """Upload file to transfer.sh service"""
        try:
            with open(file_path, 'rb') as file:
                response = requests.put(
                    f'https://transfer.sh/{filename}',
                    data=file,
                    timeout=60,
                    headers={'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'}
                )
                
            if response.status_code == 200:
                url = response.text.strip()
                return {
                    'success': True,
                    'url': url,
                    'expires': '14 days ⏰',
                    'service': 'transfer.sh 🚀',
                    'emoji': '🔗'
                }
            else:
                return {'success': False, 'error': f'HTTP {response.status_code}'}
                
        except Exception as e:
            return {'success': False, 'error': str(e)}
    
    @staticmethod
    async def upload_to_anonfiles(file_path, filename):
        """Upload file to anonfiles.com"""
        try:
            with open(file_path, 'rb') as file:
                files = {'file': (filename, file)}
                response = requests.post(
                    'https://api.anonfiles.com/upload',
                    files=files,
                    timeout=60
                )
                data = response.json()
                
            if data.get('status'):
                return {
                    'success': True,
                    'url': data['data']['file']['url']['full'],
                    'expires': 'Never 🎉',
                    'service': 'anonfiles.com 🔒',
                    'emoji': '🌐'
                }
            else:
                return {'success': False, 'error': data.get('error', {}).get('message', 'Upload failed')}
                
        except Exception as e:
            return {'success': False, 'error': str(e)}
    
    @staticmethod
    async def upload_to_filebin(file_path, filename):
        """Upload file to filebin.net"""
        try:
            with open(file_path, 'rb') as file:
                response = requests.post(
                    'https://filebin.net',
                    files={'file': (filename, file)},
                    timeout=60
                )
                
            if response.status_code == 200:
                return {
                    'success': True,
                    'url': response.url,
                    'expires': '30 days 📅',
                    'service': 'filebin.net 📁',
                    'emoji': '📦'
                }
            else:
                return {'success': False, 'error': f'HTTP {response.status_code}'}
                
        except Exception as e:
            return {'success': False, 'error': str(e)}
    
    @staticmethod
    async def create_telegram_download_info(file_data, file_type, filename, file_size):
        """Create enhanced Telegram download information"""
        download_info = (
            f"🎉 **File Successfully Received!**\n\n"
            f"📁 **File Name:** `{filename}`\n"
            f"📊 **File Size:** {DownloadProgress().format_size(file_size)}\n"
            f"🎯 **File Type:** {file_type.upper()}\n"
            f"🆔 **Telegram ID:** `{file_data.file_id}`\n\n"
            f"🔗 **Download Methods:**\n"
            f"• 📱 **Mobile**: Long press → 'Save to Downloads'\n"
            f"• 💻 **Desktop**: Right click → 'Save as'\n"
            f"• ☁️ **Cloud**: Forward to 'Saved Messages'\n"
            f"• 📤 **Export**: Share directly to other apps\n\n"
            f"💡 **Pro Tips:**\n"
            f"• Files are stored in Telegram cloud forever\n"
            f"• Access from any device with your account\n"
            f"• No size limits for Telegram storage\n"
            f"• Instant download speeds\n\n"
            f"🚀 **Your file is ready to use!**"
        )
        return download_info
    
    @staticmethod
    async def generate_public_links(file_path, filename):
        """Try multiple services to generate public download links"""
        services = [
            FileUploader.upload_to_anonfiles,  # Most reliable
            FileUploader.upload_to_fileio,
            FileUploader.upload_to_transfersh,
            FileUploader.upload_to_filebin,
        ]
        
        results = []
        for service in services:
            try:
                logger.info(f"Trying upload service: {service.__name__}")
                result = await service(file_path, filename)
                if result['success']:
                    logger.info(f"Upload successful to {service.__name__}")
                    results.append(result)
                    # Return first successful result
                    return results
                else:
                    logger.warning(f"Upload failed to {service.__name__}: {result.get('error')}")
            except Exception as e:
                logger.error(f"Upload service error {service.__name__}: {e}")
                continue
        
        return results

class HardsubBot:
    def __init__(self, token):
        self.token = token
        self.application = Application.builder().token(token).build()
        self.setup_handlers()
        self.load_data()
        self.download_progress = {}
        self.uploader = FileUploader()
    
    async def send_typing(self, update):
        """Send typing action"""
        try:
            await self.application.bot.send_chat_action(
                chat_id=update.effective_chat.id, 
                action="typing"
            )
            await asyncio.sleep(1)
        except:
            pass
    
    async def set_bot_commands(self):
        """Set bot commands for the menu"""
        commands = [
            ("start", "🎉 Start your file adventure!"),
            ("help", "📚 Complete user guide"),
            ("download", "📥 Get public download links"),
            ("large", "🎬 Process large videos (50-300MB)"),
            ("huge", "💪 Process huge videos (300MB-1GB)"),
            ("giant", "🚀 Process giant videos (1GB-2GB)"),
            ("status", "📊 Check your stats & bot status"),
            ("features", "🌟 See all amazing features"),
        ]
        
        # Set commands for bot menu
        await self.application.bot.set_my_commands(commands)
    
    def load_data(self):
        """Load banned users and stats from file"""
        try:
            if os.path.exists('bot_data.json'):
                with open('bot_data.json', 'r') as f:
                    data = json.load(f)
                    global BANNED_USERS, USER_STATS
                    BANNED_USERS = set(data.get('banned_users', []))
                    USER_STATS = data.get('user_stats', {})
        except Exception as e:
            logger.error(f"Error loading data: {e}")
    
    def save_data(self):
        """Save banned users and stats to file"""
        try:
            data = {
                'banned_users': list(BANNED_USERS),
                'user_stats': USER_STATS
            }
            with open('bot_data.json', 'w') as f:
                json.dump(data, f)
        except Exception as e:
            logger.error(f"Error saving data: {e}")
    
    def is_admin(self, user_id):
        """Check if user is admin"""
        return user_id in ADMIN_IDS
    
    def is_banned(self, user_id):
        """Check if user is banned"""
        return user_id in BANNED_USERS
    
    def update_stats(self, user_id, action, file_size=0):
        """Update user statistics"""
        if user_id not in USER_STATS:
            USER_STATS[user_id] = {
                'videos_processed': 0,
                'files_downloaded': 0,
                'total_size_processed': 0,
                'last_activity': None,
                'join_date': time.time()
            }
        
        if action == 'video_processed':
            USER_STATS[user_id]['videos_processed'] += 1
        elif action == 'file_downloaded':
            USER_STATS[user_id]['files_downloaded'] += 1
            
        if file_size > 0:
            USER_STATS[user_id]['total_size_processed'] += file_size
            
        USER_STATS[user_id]['last_activity'] = time.time()
        self.save_data()
    
    async def check_access(self, update: Update):
        """Check if user has access to use the bot"""
        user_id = update.effective_user.id
        
        if self.is_banned(user_id):
            await update.message.reply_text(
                "🚫 **Access Denied**\n\n"
                "You have been banned from using this bot.\n"
                "Contact admin if you think this is a mistake."
            )
            return False
        
        return True
    
    def setup_handlers(self):
        # Public commands
        self.application.add_handler(CommandHandler("start", self.start_command))
        self.application.add_handler(CommandHandler("help", self.help_command))
        self.application.add_handler(CommandHandler("features", self.features_command))
        self.application.add_handler(CommandHandler("large", self.large_command))
        self.application.add_handler(CommandHandler("huge", self.huge_command))
        self.application.add_handler(CommandHandler("giant", self.giant_command))
        self.application.add_handler(CommandHandler("status", self.status_command))
        self.application.add_handler(CommandHandler("commands", self.commands_command))
        self.application.add_handler(CommandHandler("download", self.download_command))
        
        # Admin commands
        self.application.add_handler(CommandHandler("admin", self.admin_command))
        self.application.add_handler(CommandHandler("ban", self.ban_command))
        self.application.add_handler(CommandHandler("unban", self.unban_command))
        self.application.add_handler(CommandHandler("stats", self.stats_command))
        self.application.add_handler(CommandHandler("users", self.users_command))
        self.application.add_handler(CommandHandler("broadcast", self.broadcast_command))
        
        # Message handlers
        self.application.add_handler(MessageHandler(filters.VIDEO, self.handle_video))
        self.application.add_handler(MessageHandler(filters.Document.VIDEO, self.handle_video_document))
        self.application.add_handler(MessageHandler(filters.Document.ALL, self.handle_document))
        self.application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, self.handle_text))
    
    # ===== COMMAND HANDLERS =====
    
    async def start_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Start command handler"""
        if not await self.check_access(update):
            return
        
        await self.send_typing(update)
        
        welcome_text = (
            "🤖 **Welcome to FileMaster Pro!** 🚀\n\n"
            
            "✨ **What I Can Do:**\n"
            "• 📥 Generate **PUBLIC** download links\n"
            "• 🎬 Add hardcoded subtitles to videos\n" 
            "• 💾 Handle files up to **2GB**\n"
            "• 🚀 Fast processing with progress tracking\n"
            "• 🌐 Multiple upload services\n\n"
            
            "🛠️ **Quick Start:**\n"
            "• Send any file for download links 📁\n"
            "• Send video + subtitle for processing 🎬\n"
            "• Use /download for public links 🌐\n\n"
            
            "📋 **Popular Commands:**\n"
            "/download - Get public download links\n"
            "/features - See all amazing features\n" 
            "/help - Complete guide\n"
            "/status - Your statistics\n\n"
            
            "🚀 **Ready when you are! Send me a file or use /download**"
        )
        
        await update.message.reply_text(welcome_text, parse_mode=ParseMode.MARKDOWN)

    async def help_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Help command handler"""
        if not await self.check_access(update):
            return
        
        await self.send_typing(update)
        
        help_text = (
            "📚 **Complete User Guide**\n\n"
            
            "🎯 **Getting Public Download Links:**\n"
            "1. Use `/download` command\n" 
            "2. Send any file (video, document, image, etc.)\n"
            "3. I'll upload to multiple services\n"
            "4. Get PUBLIC shareable links! 🌐\n\n"
            
            "🎬 **Video Subtitle Processing:**\n"
            "1. Send video file (or use /large, /huge, /giant)\n"
            "2. Send .srt subtitle file\n"
            "3. Get video with permanent subtitles!\n\n"
            
            "📁 **File Size Limits:**\n"
            "• Public links: Up to 500MB\n"
            "• Direct upload: Up to 50MB (Telegram limit)\n"
            "• /large: 50-300MB via URLs\n"
            "• /huge: 300MB-1GB via URLs\n"
            "• /giant: 1GB-2GB via URLs\n\n"
            
            "🔄 **Smart Modes:**\n"
            "• **Auto-detect**: Send file → Choose action\n"
            "• **Download mode**: /download → Public links\n"
            "• **Process mode**: Send video → Add subtitles\n\n"
            
            "💡 **Pro Tips:**\n"
            "• Use /download for shareable links\n"
            "• Large files work better with URLs\n"
            "• Check /status for your statistics\n"
            "• Use /features to see all capabilities\n\n"
            
            "🚀 **Need help? Just send a file and I'll guide you!**"
        )
        
        await update.message.reply_text(help_text, parse_mode=ParseMode.MARKDOWN)

    async def features_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Show all features"""
        if not await self.check_access(update):
            return
        
        await self.send_typing(update)
        
        features_text = (
            "🌟 **FileMaster Pro - All Features**\n\n"
            
            "📥 **Download Features:**\n"
            "• Public download links (anonfiles.com, file.io, transfer.sh)\n" 
            "• File info & details\n"
            "• Progress tracking\n"
            "• Multiple service fallbacks\n"
            "• Up to 500MB file support\n\n"
            
            "🎬 **Video Processing:**\n"
            "• Hardcoded subtitle addition\n"
            "• Support for MP4, MKV, AVI, MOV\n"
            "• SRT subtitle support\n"
            "• Large file handling (up to 2GB)\n"
            "• Professional FFmpeg processing\n\n"
            
            "⚡ **Performance:**\n"
            "• Real-time progress updates\n"
            "• System monitoring\n"
            "• Fast upload/download speeds\n"
            "• Automatic file cleanup\n"
            "• Error recovery system\n\n"
            
            "👤 **User Experience:**\n"
            "• Smart mode detection\n"
            "• Interactive responses\n"
            "• Statistics tracking\n"
            "• Beautiful formatting\n"
            "• Clear instructions\n\n"
            
            "🛡️ **Admin Features:**\n"
            "• User management\n"
            "• System statistics\n"
            "• Broadcast messages\n"
            "• Access control\n\n"
            
            "🚀 **Ready to experience the ultimate file bot?**\n"
            "Just send me a file or use /download!"
        )
        
        await update.message.reply_text(features_text, parse_mode=ParseMode.MARKDOWN)

    async def large_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Large files command handler"""
        if not await self.check_access(update):
            return
        
        await self.send_typing(update)
        
        # Set mode to video processing
        context.user_data['mode'] = 'video_processing'
        
        await update.message.reply_text(
            "🎬 **Large Video Processing (50-300MB)**\n\n"
            "📹 Please send the video URL:\n"
            "• Google Drive, Dropbox, Direct links\n"
            "• Then send the .srt subtitle file\n\n"
            "⚡ I'll handle the rest!",
            parse_mode=ParseMode.MARKDOWN
        )

    async def huge_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Huge files command handler"""
        if not await self.check_access(update):
            return
        
        await self.send_typing(update)
        
        # Set mode to video processing
        context.user_data['mode'] = 'video_processing'
        
        await update.message.reply_text(
            "💪 **Huge Video Processing (300MB-1GB)**\n\n"
            "📹 Please send the video URL:\n"
            "• Google Drive, Dropbox, Direct links\n"
            "• Then send the .srt subtitle file\n\n"
            "🚀 Powering up for big files!",
            parse_mode=ParseMode.MARKDOWN
        )

    async def giant_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Giant files command handler"""
        if not await self.check_access(update):
            return
        
        await self.send_typing(update)
        
        # Set mode to video processing
        context.user_data['mode'] = 'video_processing'
        
        await update.message.reply_text(
            "🚀 **Giant Video Processing (1GB-2GB)**\n\n"
            "📹 Please send the video URL:\n"
            "• Google Drive, Dropbox, Direct links\n"
            "• Then send the .srt subtitle file\n\n"
            "🦾 Maximum power engaged!",
            parse_mode=ParseMode.MARKDOWN
        )

    async def status_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Status command handler"""
        if not await self.check_access(update):
            return
        
        await self.send_typing(update)
        
        user_id = update.effective_user.id
        stats = USER_STATS.get(user_id, {})
        
        status_text = (
            "📊 **Your Personal Dashboard**\n\n"
            f"👤 **User Stats:**\n"
            f"• 🎬 Videos Processed: {stats.get('videos_processed', 0)}\n"
            f"• 📁 Files Downloaded: {stats.get('files_downloaded', 0)}\n"
            f"• 💾 Total Size: {DownloadProgress().format_size(stats.get('total_size_processed', 0))}\n"
            f"• 🕒 Last Active: {datetime.fromtimestamp(stats.get('last_activity', time.time())).strftime('%Y-%m-%d %H:%M:%S') if stats.get('last_activity') else 'Never'}\n"
            f"• 📅 Join Date: {datetime.fromtimestamp(stats.get('join_date', time.time())).strftime('%Y-%m-%d %H:%M:%S')}\n\n"
            
            f"🤖 **Bot Status:**\n"
            f"• 🟢 Status: Operational\n"
            f"• 🕒 Uptime: {DownloadProgress().format_time(time.time() - START_TIME)}\n"
            f"• 🌐 Public Links: Available\n"
            f"• 🎯 Ready for action!\n\n"
            
            "🚀 **Keep up the great work!**"
        )
        
        await update.message.reply_text(status_text, parse_mode=ParseMode.MARKDOWN)

    async def commands_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Commands list handler"""
        if not await self.check_access(update):
            return
        
        await self.send_typing(update)
        
        commands_text = (
            "📋 **Command Center**\n\n"
            
            "🎯 **Main Commands:**\n"
            "/download - Get PUBLIC download links 🌐\n"
            "/features - See all amazing features 🌟\n"
            "/help - Complete user guide 📚\n"
            "/status - Your personal dashboard 📊\n\n"
            
            "🎬 **Video Processing:**\n" 
            "/large - Process 50-300MB files 🎬\n"
            "/huge - Process 300MB-1GB files 💪\n"
            "/giant - Process 1GB-2GB files 🚀\n\n"
            
            "🛡️ **Admin Commands:**\n"
            "/admin - Admin panel\n"
            "/stats - System statistics\n"
            "/users - User management\n\n"
            
            "💡 **Quick Tip:** Just send me a file to get started!"
        )
        
        await update.message.reply_text(commands_text, parse_mode=ParseMode.MARKDOWN)
    
    async def download_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Generate download links for files"""
        if not await self.check_access(update):
            return
        
        await self.send_typing(update)
        
        # Set mode to download link generation
        context.user_data['mode'] = 'download_link'
        
        help_text = (
            "📥 **PUBLIC Download Link Generator**\n\n"
            
            "🚀 **How It Works:**\n"
            "1. Send me any file (video, document, image, etc.)\n"
            "2. I'll upload to multiple file hosting services\n"
            "3. Get PUBLIC shareable download links\n"
            "4. Share with anyone - works everywhere! 🌐\n\n"
            
            "✨ **Features:**\n"
            "• Multiple hosting services for reliability\n"
            "• File size support up to 500MB\n"
            "• Fast upload speeds\n"
            "• Progress tracking\n"
            "• No registration required\n\n"
            
            "📁 **Supported Files:**\n"
            "• Videos (MP4, MKV, AVI, etc.)\n"
            "• Documents (PDF, DOC, TXT, etc.)\n"
            "• Images (JPG, PNG, GIF, etc.)\n"
            "• Audio files (MP3, WAV, etc.)\n"
            "• Any file type! 🎉\n\n"
            
            "🌐 **Ready to create public links? Send me any file!**"
        )
        
        await update.message.reply_text(help_text, parse_mode=ParseMode.MARKDOWN)
    
    # ===== ADMIN COMMANDS =====
    
    async def admin_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Admin panel command"""
        user_id = update.effective_user.id
        if not self.is_admin(user_id):
            await update.message.reply_text("❌ Access denied.")
            return
        
        await self.send_typing(update)
        
        admin_text = (
            "🛡️ **Admin Control Panel**\n\n"
            "⚡ **Available Commands:**\n"
            "/ban [user_id] - Ban a user\n"
            "/unban [user_id] - Unban a user\n"
            "/stats - System statistics\n"
            "/users - User list management\n"
            "/broadcast [message] - Broadcast message\n\n"
            f"📊 **Current Stats:**\n"
            f"• 👥 Total Users: {len(USER_STATS)}\n"
            f"• 🚫 Banned Users: {len(BANNED_USERS)}\n"
            f"• 🕒 Bot Uptime: {DownloadProgress().format_time(time.time() - START_TIME)}\n"
            f"• 🌐 Public Links: Active\n\n"
            "🔧 **System Status:** 🟢 All Systems Operational"
        )
        
        await update.message.reply_text(admin_text, parse_mode=ParseMode.MARKDOWN)
    
    async def ban_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Ban user command"""
        user_id = update.effective_user.id
        if not self.is_admin(user_id):
            await update.message.reply_text("❌ Access denied.")
            return
        
        if not context.args:
            await update.message.reply_text("Usage: /ban [user_id]")
            return
        
        try:
            target_id = int(context.args[0])
            BANNED_USERS.add(target_id)
            self.save_data()
            await update.message.reply_text(f"✅ User {target_id} has been banned.")
        except ValueError:
            await update.message.reply_text("❌ Invalid user ID.")
    
    async def unban_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Unban user command"""
        user_id = update.effective_user.id
        if not self.is_admin(user_id):
            await update.message.reply_text("❌ Access denied.")
            return
        
        if not context.args:
            await update.message.reply_text("Usage: /unban [user_id]")
            return
        
        try:
            target_id = int(context.args[0])
            if target_id in BANNED_USERS:
                BANNED_USERS.remove(target_id)
                self.save_data()
                await update.message.reply_text(f"✅ User {target_id} has been unbanned.")
            else:
                await update.message.reply_text("❌ User is not banned.")
        except ValueError:
            await update.message.reply_text("❌ Invalid user ID.")
    
    async def stats_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Bot statistics command"""
        user_id = update.effective_user.id
        if not self.is_admin(user_id):
            await update.message.reply_text("❌ Access denied.")
            return
        
        await self.send_typing(update)
        
        total_videos = sum(stats.get('videos_processed', 0) for stats in USER_STATS.values())
        total_files = sum(stats.get('files_downloaded', 0) for stats in USER_STATS.values())
        active_users = len([uid for uid, stats in USER_STATS.items() 
                          if time.time() - stats.get('last_activity', 0) < 7 * 24 * 3600])
        
        stats_text = (
            "📈 **System Statistics**\n\n"
            f"👥 **User Statistics:**\n"
            f"• Total Users: {len(USER_STATS)}\n"
            f"• Active Users (7 days): {active_users}\n"
            f"• Banned Users: {len(BANNED_USERS)}\n"
            f"• Total Videos Processed: {total_videos}\n"
            f"• Total Files Downloaded: {total_files}\n\n"
            
            f"🤖 **Bot Information:**\n"
            f"• Uptime: {DownloadProgress().format_time(time.time() - START_TIME)}\n"
            f"• Admin ID: {ADMIN_IDS[0]}\n"
            f"• Public Links: ✅ Enabled\n"
            f"• System Status: 🟢 Operational\n\n"
            
            "🚀 **Performance:** Excellent"
        )
        
        await update.message.reply_text(stats_text, parse_mode=ParseMode.MARKDOWN)
    
    async def users_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Users list command"""
        user_id = update.effective_user.id
        if not self.is_admin(user_id):
            await update.message.reply_text("❌ Access denied.")
            return
        
        if not USER_STATS:
            await update.message.reply_text("No users yet.")
            return
        
        users_text = "👥 **Registered Users**\n\n"
        for uid, stats in list(USER_STATS.items())[:10]:  # Show first 10 users
            last_active = datetime.fromtimestamp(stats.get('last_activity', time.time())).strftime('%Y-%m-%d') if stats.get('last_activity') else 'Never'
            users_text += f"• ID: {uid} | Videos: {stats.get('videos_processed', 0)} | Last: {last_active}\n"
        
        if len(USER_STATS) > 10:
            users_text += f"\n... and {len(USER_STATS) - 10} more users"
        
        await update.message.reply_text(users_text, parse_mode=ParseMode.MARKDOWN)
    
    async def broadcast_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Broadcast message command"""
        user_id = update.effective_user.id
        if not self.is_admin(user_id):
            await update.message.reply_text("❌ Access denied.")
            return
        
        if not context.args:
            await update.message.reply_text("Usage: /broadcast [message]")
            return
        
        message = ' '.join(context.args)
        success_count = 0
        fail_count = 0
        
        progress_msg = await update.message.reply_text("📢 Starting broadcast...")
        
        for user_id in USER_STATS.keys():
            try:
                await context.bot.send_message(
                    chat_id=user_id, 
                    text=f"📢 **Announcement**\n\n{message}",
                    parse_mode=ParseMode.MARKDOWN
                )
                success_count += 1
                await asyncio.sleep(0.1)  # Rate limiting
                
                # Update progress every 10 messages
                if success_count % 10 == 0:
                    await progress_msg.edit_text(
                        f"📢 Broadcasting...\n"
                        f"✅ Sent: {success_count}\n"
                        f"❌ Failed: {fail_count}"
                    )
                    
            except Exception as e:
                fail_count += 1
        
        await progress_msg.edit_text(
            f"🎉 **Broadcast Completed!**\n\n"
            f"✅ Successful: {success_count}\n"
            f"❌ Failed: {fail_count}\n"
            f"📊 Total: {len(USER_STATS)} users"
        )
    
    # ===== MESSAGE HANDLERS =====
    
    async def handle_video(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Handle native video files"""
        try:
            if not await self.check_access(update):
                return
            
            user_id = update.effective_user.id
            
            # Check if user is in download mode
            if context.user_data.get('mode') == 'download_link':
                await self.generate_public_download_links(update, context)
                return
            
            # Otherwise, treat as video for processing
            if 'user_data' not in context.bot_data:
                context.bot_data['user_data'] = {}
            
            video = update.message.video
            if video is None:
                await update.message.reply_text("❌ Error: Could not process video file.")
                return
            
            context.bot_data['user_data'][user_id] = {
                'video': {
                    'file_id': video.file_id,
                    'file_size': video.file_size,
                    'type': 'native_video'
                }
            }
            
            response_text = (
                "🎬 **Video Received!** ✅\n\n"
                "📹 Now please send the **.srt subtitle file**\n\n"
                "💡 I'll add permanent subtitles to your video!"
            )
            
            await update.message.reply_text(response_text, parse_mode=ParseMode.MARKDOWN)
            
        except Exception as e:
            logger.error(f"Error handling video: {e}")
            await update.message.reply_text("❌ Error processing video. Please try again.")
    
    async def handle_video_document(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Handle video files sent as documents"""
        try:
            if not await self.check_access(update):
                return
            
            user_id = update.effective_user.id
            
            # Check if user is in download mode
            if context.user_data.get('mode') == 'download_link':
                await self.generate_public_download_links(update, context)
                return
            
            # Otherwise, treat as video for processing
            if 'user_data' not in context.bot_data:
                context.bot_data['user_data'] = {}
            
            document = update.message.document
            if document is None:
                await update.message.reply_text("❌ Error: Could not process video file.")
                return
            
            # Check if it's a video file by mime type or file extension
            file_name = document.file_name or ""
            file_extension = file_name.split('.')[-1].lower() if '.' in file_name else ""
            mime_type = document.mime_type or ""
            
            is_video = any(ext in file_extension for ext in ['mp4', 'mkv', 'avi', 'mov', 'wmv', 'flv']) or 'video' in mime_type
            
            if not is_video:
                await update.message.reply_text("❌ Please send a video file.")
                return
            
            context.bot_data['user_data'][user_id] = {
                'video': {
                    'file_id': document.file_id,
                    'file_size': document.file_size,
                    'file_name': file_name,
                    'type': 'document_video'
                }
            }
            
            response_text = (
                "🎬 **Video Received!** ✅\n\n"
                "📹 Now please send the **.srt subtitle file**\n\n"
                "💡 I'll add permanent subtitles to your video!"
            )
            
            await update.message.reply_text(response_text, parse_mode=ParseMode.MARKDOWN)
            
        except Exception as e:
            logger.error(f"Error handling video document: {e}")
            await update.message.reply_text("❌ Error processing video file.")
    
    async def handle_document(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Handle document files (subtitles and other files)"""
        try:
            user_id = update.effective_user.id
            
            # Check if user is in download mode
            if context.user_data.get('mode') == 'download_link':
                await self.generate_public_download_links(update, context)
                return
            
            # Check if this is a subtitle file for video processing
            if ('user_data' in context.bot_data and 
                user_id in context.bot_data['user_data'] and 
                'video' in context.bot_data['user_data'][user_id]):
                
                document = update.message.document
                if document is None:
                    await update.message.reply_text("❌ Error: Could not process subtitle file.")
                    return
                
                file_name = document.file_name or ""
                file_extension = file_name.split('.')[-1].lower() if '.' in file_name else ""
                
                if file_extension != 'srt':
                    await update.message.reply_text("❌ Please send an .srt subtitle file.")
                    return
                
                user_data = context.bot_data['user_data'][user_id]
                user_data['subtitle'] = {
                    'file_id': document.file_id,
                    'file_name': file_name
                }
                
                await self.process_hardsub(update, context, user_id)
                return
            
            # If no video is waiting and not in download mode, show instructions
            await update.message.reply_text(
                "🤔 **What would you like to do?**\n\n"
                "📁 **For Public Download Links:**\n"
                "Use `/download` command or send any file\n\n"
                "🎬 **For Video Processing:**\n"
                "Send a video file first, then subtitle\n\n"
                "💡 **Quick Tip:** Use /download for shareable links!",
                parse_mode=ParseMode.MARKDOWN
            )
            
        except Exception as e:
            logger.error(f"Error handling document: {e}")
            await update.message.reply_text("❌ Error processing file.")
    
    async def handle_text(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Handle text messages (URLs)"""
        try:
            if not await self.check_access(update):
                return
            
            await self.send_typing(update)
            
            text = update.message.text.strip()
            user_id = update.effective_user.id
            
            # Check if it's a URL
            if text.startswith(('http://', 'https://')):
                # Initialize user data if not exists
                if 'user_data' not in context.bot_data:
                    context.bot_data['user_data'] = {}
                
                if user_id not in context.bot_data['user_data']:
                    context.bot_data['user_data'][user_id] = {}
                
                context.bot_data['user_data'][user_id]['video_url'] = text
                
                await update.message.reply_text(
                    "🌐 **Video URL Received!** ✅\n\n"
                    "📹 Now please send the **.srt subtitle file**\n\n"
                    "💡 I'll download and process your video!",
                    parse_mode=ParseMode.MARKDOWN
                )
            else:
                await update.message.reply_text(
                    "🤖 **How can I help you?**\n\n"
                    "• Send a file for processing 📁\n"
                    "• Use `/download` for public links 🌐\n"
                    "• Send a video URL for processing 🎬\n"
                    "• Use `/help` for complete guide 📚",
                    parse_mode=ParseMode.MARKDOWN
                )
                
        except Exception as e:
            logger.error(f"Error handling text: {e}")
            await update.message.reply_text(
                "❌ Error processing URL."
            )
    
    # ===== DOWNLOAD LINK METHODS =====
    
    async def generate_public_download_links(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        """Generate PUBLIC download links for uploaded file"""
        try:
            user_id = update.effective_user.id
            
            # Determine file type and get file object
            file = None
            file_type = "unknown"
            
            if update.message.video:
                file = update.message.video
                file_type = "video"
            elif update.message.document:
                file = update.message.document
                file_type = "document"
            elif update.message.audio:
                file = update.message.audio
                file_type = "audio"
            elif update.message.photo:
                file = update.message.photo[-1]  # Highest resolution
                file_type = "photo"
            else:
                await update.message.reply_text("❌ Unsupported file type")
                return
            
            if file is None:
                await update.message.reply_text("❌ Could not process file.")
                return
            
            # Get file info
            file_name = getattr(file, 'file_name', f'{file_type}_{file.file_id}')
            file_size = getattr(file, 'file_size', 0)
            
            # Check file size (limit to 500MB for public uploads)
            if file_size > 500 * 1024 * 1024:  # 500MB limit
                telegram_info = await self.uploader.create_telegram_download_info(
                    file, file_type, file_name, file_size
                )
                await update.message.reply_text(telegram_info, parse_mode=ParseMode.MARKDOWN)
                self.update_stats(user_id, 'file_downloaded', file_size)
                return
            
            # Send initial message
            progress_msg = await update.message.reply_text(
                "📤 **Creating Public Download Links...**\n\n"
                f"📁 **File:** `{file_name}`\n"
                f"📊 **Size:** {DownloadProgress().format_size(file_size)}\n"
                f"🎯 **Type:** {file_type.upper()}\n\n"
                "🔄 Preparing upload...",
                parse_mode=ParseMode.MARKDOWN
            )
            
            # Download file from Telegram
            temp_path = await self.download_file(context, file.file_id, 'upload')
            
            try:
                # Generate public links
                await progress_msg.edit_text(
                    "📤 **Uploading to Public Services...**\n\n"
                    f"📁 **File:** `{file_name}`\n"
                    f"📊 **Size:** {DownloadProgress().format_size(file_size)}\n"
                    f"🎯 **Type:** {file_type.upper()}\n\n"
                    "🌐 Trying multiple services...",
                    parse_mode=ParseMode.MARKDOWN
                )
                
                upload_results = await self.uploader.generate_public_links(temp_path, file_name)
                
                if upload_results:
                    # Success - show public links
                    success_text = (
                        "✅ **PUBLIC DOWNLOAD LINKS READY!** 🌐\n\n"
                        f"📁 **File:** `{file_name}`\n"
                        f"📊 **Size:** {DownloadProgress().format_size(file_size)}\n"
                        f"🎯 **Type:** {file_type.upper()}\n\n"
                        "🔗 **PUBLIC DOWNLOAD LINKS:**\n\n"
                    )
                    
                    for result in upload_results:
                        success_text += f"• {result['emoji']} **{result['service']}**\n"
                        success_text += f"  📎 {result['url']}\n"
                        success_text += f"  ⏰ Expires: {result['expires']}\n\n"
                    
                    success_text += (
                        "💡 **Share these links with anyone!**\n"
                        "They work in browsers, apps, everywhere! 🌍\n\n"
                        "🚀 **Happy sharing!**"
                    )
                    
                    await progress_msg.edit_text(success_text, parse_mode=ParseMode.MARKDOWN)
                    self.update_stats(user_id, 'file_downloaded', file_size)
                    
                else:
                    # All upload services failed - provide enhanced Telegram info
                    telegram_info = await self.uploader.create_telegram_download_info(
                        file, file_type, file_name, file_size
                    )
                    await progress_msg.edit_text(telegram_info, parse_mode=ParseMode.MARKDOWN)
                    self.update_stats(user_id, 'file_downloaded', file_size)
                
            except Exception as upload_error:
                logger.error(f"Upload error: {upload_error}")
                # Fallback to Telegram download info
                telegram_info = await self.uploader.create_telegram_download_info(
                    file, file_type, file_name, file_size
                )
                await progress_msg.edit_text(telegram_info, parse_mode=ParseMode.MARKDOWN)
                self.update_stats(user_id, 'file_downloaded', file_size)
            
            finally:
                # Cleanup temp file
                self.cleanup_files([temp_path])
                
        except Exception as e:
            logger.error(f"Download link error: {e}")
            await update.message.reply_text(
                "❌ Error processing file.\n"
                "💡 Please try again with a different file."
            )
    
    async def download_file(self, context: ContextTypes.DEFAULT_TYPE, file_id: str, file_type: str) -> str:
        """Download file from Telegram"""
        try:
            file = await context.bot.get_file(file_id)
            
            with tempfile.NamedTemporaryFile(delete=False, suffix=f'_{file_type}') as temp_file:
                temp_path = temp_file.name
            
            await file.download_to_drive(temp_path)
            return temp_path
            
        except Exception as e:
            logger.error(f"Error downloading {file_type}: {e}")
            raise
    
    def cleanup_files(self, file_paths: list):
        """Clean up temporary files"""
        for file_path in file_paths:
            try:
                if os.path.exists(file_path):
                    os.unlink(file_path)
            except Exception as e:
                logger.error(f"Error cleaning up {file_path}: {e}")
    
    # ===== PROCESSING METHODS =====
    
    async def process_hardsub(self, update: Update, context: ContextTypes.DEFAULT_TYPE, user_id: int):
        """Process video with hardcoded subtitles"""
        try:
            progress_msg = await update.message.reply_text(
                "🔄 **Starting Video Processing...**\n\n"
                "⚡ This may take a few minutes\n"
                "📊 Progress updates coming soon!",
                parse_mode=ParseMode.MARKDOWN
            )
            
            # Get user data
            user_data = context.bot_data['user_data'][user_id]
            
            # Download video and subtitle files
            if 'video_url' in user_data:
                video_url = user_data['video_url']
                video_path = await self.download_from_url(video_url, user_id, "video")
            else:
                video_path = await self.download_file(context, user_data['video']['file_id'], 'video')
            
            subtitle_path = await self.download_file(context, user_data['subtitle']['file_id'], 'subtitle')
            
            # Process video with FFmpeg
            output_path = await self.add_subtitles_with_ffmpeg(video_path, subtitle_path)
            
            # Send processed video
            with open(output_path, 'rb') as video_file:
                await update.message.reply_video(
                    video=video_file,
                    caption=(
                        "✅ **Video Processing Complete!**\n\n"
                        "🎬 Subtitles have been permanently added to your video!\n"
                        "💾 Ready to download and share!"
                    ),
                    parse_mode=ParseMode.MARKDOWN
                )
            
            # Update statistics
            self.update_stats(user_id, 'video_processed')
            
            # Cleanup
            self.cleanup_files([video_path, subtitle_path, output_path])
            
            # Clear user data
            if user_id in context.bot_data['user_data']:
                del context.bot_data['user_data'][user_id]
            
        except Exception as e:
            logger.error(f"Error in process_hardsub: {e}")
            await update.message.reply_text(
                "❌ Error processing video. Please try again.\n"
                "💡 Make sure your files are valid and try again."
            )
            
            # Cleanup on error
            if 'user_data' in context.bot_data and user_id in context.bot_data['user_data']:
                del context.bot_data['user_data'][user_id]
    
    async def download_from_url(self, url: str, user_id: int, filename: str) -> str:
        """Download file from URL"""
        try:
            with tempfile.NamedTemporaryFile(delete=False, suffix=f'_{filename}') as temp_file:
                temp_path = temp_file.name
            
            response = requests.get(url, stream=True, timeout=1800)
            response.raise_for_status()
            
            with open(temp_path, 'wb') as f:
                for chunk in response.iter_content(chunk_size=8192):
                    if chunk:
                        f.write(chunk)
            
            return temp_path
            
        except Exception as e:
            logger.error(f"Error downloading from URL: {e}")
            raise
    
    async def add_subtitles_with_ffmpeg(self, video_path: str, subtitle_path: str) -> str:
        """Add subtitles to video using FFmpeg"""
        try:
            output_path = video_path + '_hardsub.mp4'
            
            cmd = [
                'ffmpeg', '-i', video_path,
                '-vf', f"subtitles={subtitle_path}",
                '-c:a', 'copy',
                '-y',
                output_path
            ]
            
            process = await asyncio.create_subprocess_exec(
                *cmd,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE
            )
            
            stdout, stderr = await process.communicate()
            
            if process.returncode != 0:
                logger.error(f"FFmpeg error: {stderr.decode()}")
                raise Exception("FFmpeg processing failed")
            
            return output_path
            
        except Exception as e:
            logger.error(f"Error in add_subtitles_with_ffmpeg: {e}")
            raise

    async def run_async(self):
        """Run the bot asynchronously"""
        print("🤖 FileMaster Pro starting...")
        print("📥 PUBLIC Download links enabled")
        print("🚀 Multiple upload services ready")
        print("🛡️ Admin ID: (admin id")
        print("🎬 Supports files up to 2GB")
        print("🌐 Upload services: anonfiles.com, file.io, transfer.sh, filebin.net")
        
        # Set bot commands asynchronously
        await self.set_bot_commands()
        print("✅ Bot commands set successfully")
        
        print("🎯 Bot is now LIVE and ready!")
        print("Press Ctrl+C to stop")
        
        # Start the bot
        await self.application.initialize()
        await self.application.start()
        await self.application.updater.start_polling()
        
        # Keep the bot running
        while True:
            await asyncio.sleep(3600)
    
    def run(self):
        """Run the bot (synchronous wrapper for async)"""
        asyncio.run(self.run_async())

def main():
    BOT_TOKEN = "(token to be added "
    
    bot = HardsubBot(BOT_TOKEN)
    bot.run()

if __name__ == '__main__':
    main()
