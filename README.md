# video-compare-wrapper
[video-compare](https://github.com/pixop/video-compare/) tool wrapper, adding support for drag'n'drop from SMB drives on KDE systems. Made from the [own issue](https://github.com/pixop/video-compare/issues/131#issuecomment-3863643218)

# About
It's just simple wrapper to add the SMB support. Details about how to do that is pending

Basically you will need to or compile the video-compare from source with the right FFmpeg config, or compile FFmpeg from source, installing the video-compare from brew and editing PATH to add the own build of FFmpeg, or... (There's a lot of configurations how to do that, I'll leave what I did - unideal but works)

1. Install video-compare from brew: ``
2. Get an latest source of FFmpeg (warning! you may need specific versions of libs that included in FFmpeg, you will see what exactly on first attempt to run the video-compare after all the things done): `wget https://ffmpeg.org/releases/ffmpeg-8.0.1.tar.xz && tar xf ffmpeg-8.0.1.tar.xz && cd ffmpeg-8.0.1`
3. Configure the FFmpeg with `--enable-libsmbclient` option to enable SMB support: `./configure     --enable-libsmbclient     --enable-gpl     --enable-version3     --enable-shared     --prefix=/usr/local`
4. Compile FFmpeg with `make -j$(nproc)`
5. Install it: `sudo make install`
7. ~~Copy to bin folder~~
8. Add new FFmpeg build to the PATH: `(automatically handled? need a test on fresh system to avoid weird things to happen)`
9. ~~Delete FFmpeg from dependencies in brew (actually just change the PATH~~
10. Try to run video-compare

If after everything that was made, video-compare complaints about the missing libs - do from the 2nd to 10th steps again, but use the FFmpeg version that comes with those libs

The "complete" (untested) instruction:

# Video-Compare Setup Guide for KDE Neon

This guide shows how to set up `video-compare` with SMB network share support and Dolphin file manager integration on KDE Neon (Ubuntu-based systems).

## Why This Setup?

- **FFmpeg with SMB**: Allows direct access to videos on network shares (like `smb://server/share/video.mkv`)
- **KIO-fuse integration**: Works with KDE's native file mounting system
- **Dolphin context menu**: Compare two videos by right-clicking them in the file manager

## Prerequisites

- KDE Neon or Ubuntu-based system
- Basic terminal knowledge
- Network share accessible via SMB

---

## Part 1: Install Dependencies

```bash
# Update package list
sudo apt update

# Install build tools and libraries
sudo apt install -y build-essential yasm nasm pkg-config \
    libsmbclient-dev libdav1d-dev wget
```

---

## Part 2: Build FFmpeg with SMB Support

### Download FFmpeg 8.0.1

```bash
cd ~/Downloads
wget https://ffmpeg.org/releases/ffmpeg-8.0.1.tar.xz
tar xf ffmpeg-8.0.1.tar.xz
cd ffmpeg-8.0.1
```

### Configure and Compile

```bash
# Configure with SMB and AV1 support
./configure \
    --enable-libsmbclient \
    --enable-gpl \
    --enable-version3 \
    --enable-shared \
    --enable-libdav1d \
    --prefix=/usr/local

# Compile (this takes 10-30 minutes)
make -j$(nproc)

# Install
sudo make install

# Update library cache
sudo ldconfig
```

### Verify Installation

```bash
# Check if SMB protocol is available
/usr/local/bin/ffmpeg -protocols 2>&1 | grep smb

# You should see:
#   smb
#   smb
```

---

## Part 3: Install video-compare

```bash
# Install via Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Add Homebrew to PATH (if not already done)
echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.bashrc
source ~/.bashrc

# Install video-compare
brew install video-compare
```

### Fix Library Dependencies

Video-compare needs to use the custom FFmpeg libraries:

```bash
# Make custom FFmpeg libraries available
echo "/usr/local/lib" | sudo tee /etc/ld.so.conf.d/ffmpeg-custom.conf
sudo ldconfig

# Verify libraries are found
ldconfig -p | grep libavformat

# You should see libavformat.so.62 in the list
```

---

## Part 4: Create Wrapper Scripts

### Create the bin directory

```bash
mkdir -p ~/bin
```

### Script 1: SMB URL Converter (`video-compare-smb`)

This converts `smb://` URLs to KIO-fuse paths.

```bash
nano ~/bin/video-compare-smb
```

Paste this content:

```bash
#!/bin/bash
# Convert smb:// URLs to KIO-fuse paths

convert_path() {
    local url="$1"
    if [[ "$url" == smb://* ]]; then
        # Find the actual kio-fuse directory
        local kio_dir=$(ls -t /run/user/$UID/ | grep '^kio-fuse-' | head -n1)
        # Remove smb:// prefix and reconstruct path
        local smb_path="${url#smb://}"
        echo "/run/user/$UID/$kio_dir/smb/$smb_path"
    else
        echo "$url"
    fi
}

ARG1=$(convert_path "$1")
ARG2=$(convert_path "$2")

echo "Left:  $ARG1"
echo "Right: $ARG2"

video-compare "$ARG1" "$ARG2"
```

Save (Ctrl+X, Y, Enter) and make executable:

```bash
chmod +x ~/bin/video-compare-smb
```

### Script 2: Dolphin Integration (`video-compare-dolphin`)

This handles multiple file selection from Dolphin.

```bash
nano ~/bin/video-compare-dolphin
```

Paste this content:

```bash
#!/bin/bash
# Dolphin video compare wrapper

# Convert all arguments
FILES=()
for arg in "$@"; do
    if [[ "$arg" == smb://* ]]; then
        kio_dir=$(ls -t /run/user/$UID/ | grep '^kio-fuse-' | head -n1)
        smb_path="${arg#smb://}"
        FILES+=("/run/user/$UID/$kio_dir/smb/$smb_path")
    else
        FILES+=("$arg")
    fi
done

# Check if we have exactly 2 files
if [ ${#FILES[@]} -ne 2 ]; then
    kdialog --error "Please select exactly 2 video files to compare"
    exit 1
fi

# Run video-compare with absolute path
/home/linuxbrew/.linuxbrew/bin/video-compare "${FILES[0]}" "${FILES[1]}"
```

Save and make executable:

```bash
chmod +x ~/bin/video-compare-dolphin
```

### Add ~/bin to PATH

```bash
# Edit bash configuration
nano ~/.bashrc

# Add this line at the end:
export PATH="$HOME/bin:$PATH"

# Save and reload
source ~/.bashrc

# Verify
which video-compare-smb
```

---

## Part 5: Dolphin Context Menu Integration

### Create Service Menu File

```bash
# Create directory if it doesn't exist
mkdir -p ~/.local/share/kio/servicemenus/

# Create the service menu
nano ~/.local/share/kio/servicemenus/video-compare.desktop
```

Paste this content:

```ini
[Desktop Entry]
Type=Service
X-KDE-ServiceTypes=KonqPopupMenu/Plugin
MimeType=video/*
Actions=compareVideos

[Desktop Action compareVideos]
Name=Compare Videos
Icon=kdenlive
Exec=/home/USERNAME/bin/video-compare-dolphin %U
```

**Important**: Replace `USERNAME` with your actual username! Or use:

```bash
# Automatically set the correct path
sed -i "s|/home/USERNAME|$HOME|" ~/.local/share/kio/servicemenus/video-compare.desktop
```

### Restart Dolphin

```bash
killall dolphin
dolphin &
```

---

## Usage

### From Terminal (SMB URLs)

```bash
# Using wrapper for SMB URLs
video-compare-smb 'smb://user@server/share/video1.mkv' 'smb://user@server/share/video2.webm'

# Using KIO-fuse paths directly
video-compare '/run/user/1000/kio-fuse-XXXXX/smb/user@server/share/video1.mkv' '/run/user/1000/kio-fuse-XXXXX/smb/user@server/share/video2.mkv'
```

### From Dolphin File Manager

1. Open Dolphin and navigate to your network share
2. Select **exactly 2 video files** (Ctrl+Click to select multiple)
3. Right-click on one of the selected files
4. Choose **"Compare Videos"** from the context menu
5. The comparison window will open automatically

---

## Troubleshooting

### Issue: "Cannot find program video-compare-smb"

**Solution**: Make sure the script is executable and in your PATH:

```bash
chmod +x ~/bin/video-compare-smb
echo $PATH | grep "$HOME/bin"
```

### Issue: "error while loading shared libraries: libavformat.so.62"

**Solution**: Update library cache:

```bash
echo "/usr/local/lib" | sudo tee /etc/ld.so.conf.d/ffmpeg-custom.conf
sudo ldconfig
ldconfig -p | grep libavformat
```

You should see `libavformat.so.62` in the output.

### Issue: "Function not implemented" when playing AV1 videos

**Solution**: Make sure libdav1d was installed and FFmpeg was configured with `--enable-libdav1d`:

```bash
sudo apt install -y libdav1d-dev
# Then rebuild FFmpeg (Part 2)
```

### Issue: SMB authentication fails

**Solution**: Include password in the URL or set up credentials file:

```bash
# Option 1: Password in URL
video-compare-smb 'smb://username:password@server/share/video.mkv' '...'

# Option 2: Create credentials file
nano ~/.smbcredentials
# Add:
# username=your_username
# password=your_password

chmod 600 ~/.smbcredentials
```

### Issue: Context menu doesn't appear in Dolphin

**Solution**: 
1. Verify the service menu file exists and has correct username
2. Restart Dolphin completely
3. Make sure you're selecting video files (the menu is filtered by MIME type)

```bash
ls -la ~/.local/share/kio/servicemenus/video-compare.desktop
killall dolphin && dolphin &
```

---

## How It Works

1. **KDE's KIO-fuse**: When you open network shares in Dolphin, KDE automatically mounts them at `/run/user/UID/kio-fuse-*/smb/`
2. **Wrapper scripts**: Convert `smb://` URLs (from drag-and-drop) to local KIO-fuse paths
3. **Custom FFmpeg**: Has libsmbclient support to handle SMB protocol directly if needed
4. **Service menu**: Integrates everything into Dolphin's right-click menu

---

## Notes

- The KIO-fuse paths are temporary and change each session
- The wrapper scripts automatically detect the current KIO-fuse mount point
- You need to keep the network share mounted in Dolphin for the comparison to work
- For best performance with large files, the KIO-fuse approach is more reliable than direct SMB access through FFmpeg

---

## Credits

- **FFmpeg**: https://ffmpeg.org/
- **video-compare**: https://github.com/pixop/video-compare
- **libsmbclient**: Part of Samba project
- **dav1d**: VideoLAN AV1 decoder

---

## License

This guide is provided as-is for educational purposes. Individual software components have their own licenses.
