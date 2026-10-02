# expo-sqlite on every platform, accepting alpha web support

User data uses upstream's schema through `expo-sqlite` on iOS, Android and web, with one shared contract-test suite. Web support in `expo-sqlite` is alpha, so an Expo upgrade may break web storage; the contract tests are how we notice. Considered and rejected: sqlite-wasm or IndexedDB on web (a second adapter and schema translation).
