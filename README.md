# VeroRun Store

Official plugin marketplace for VeroRun — browse, install, and manage extensions for your VeroRun platform.

## How It Works

This repository hosts the plugin catalog (`store_catalog.json`) and distributes plugin packages via GitHub Releases.

### For VeroRun Users

1. Open your VeroRun Admin Panel
2. Navigate to **Plugin Store**
3. Browse available plugins, install free ones, or purchase premium plugins
4. Manage licenses under **License Management**

### For Plugin Developers

Plugins are published from the private `verorun-code` repository using `publish_plugin.py`:

```bash
python tools/publish_plugin.py <plugin_identifier>
```

This script:
- Packages the plugin into a `.zip` archive
- Computes SHA256 checksum
- Uploads to GitHub Releases
- Updates `store_catalog.json`

## Repository Structure

```
verorun-store/
├── README.md
├── LICENSE
├── store_catalog.json    # Plugin catalog index
└── (GitHub Releases)     # Plugin .zip packages
```

## License

The catalog and repository files are MIT licensed. Individual plugins may have their own licensing terms.