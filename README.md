# Triv

Triv is a personal life-tracking Flutter application (sleep, habits, weight).  
It is **local-first**, with no ads and no subscription.

## Project structure

```
Triv/
├── app/                 # Flutter application (submodule)
├── apk/                 # Older release APKs (not tracked on main)
├── Triv-3.0.apk         # Latest release APK (current version)
├── README.md
└── .gitignore
```

## Getting started

The Flutter code lives in the `app/` submodule.

```bash
# Clone with submodule
git clone --recurse-submodules https://github.com/FabienRf/Triv.git

# Enter the Flutter app
cd app

# Install dependencies
flutter pub get

# Run the app
flutter run
```

## Release APK

The latest release APK (`Triv-3.0.apk`) is available at the repository root.  
Older versions are kept in the `apk/` folder.

## Features

- Sleep tracking and sleep calendar
- Diet and meal tracking
- Habit and weight tracking
- Local-first storage (no account required)
- English and French translations

## License

Private project — all rights reserved.
