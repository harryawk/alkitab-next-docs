# expo-sqlite on every platform, accepting alpha web support

User data uses upstream's schema through `expo-sqlite` on iOS, Android and web, with one shared contract-test suite. Web support in `expo-sqlite` is alpha, so an Expo upgrade may break web storage; the contract tests are how we notice. Considered and rejected: sqlite-wasm or IndexedDB on web (a second adapter and schema translation).

## Note: how the contract suite runs today

The repository takes a small `SqlDatabase` interface that `expo-sqlite`'s `SQLiteDatabase` is adapted to (`openUserDataStore`). The Jest contract suite runs the same repository against an in-memory SQLite (`sql.js`) behind that interface, so it proves the repository's SQL and rules but not `expo-sqlite`'s native or web builds. Running the suite against `expo-sqlite` on the web build (needs the WebAssembly metro setting and the `Cross-Origin-Opener-Policy` and `Cross-Origin-Embedder-Policy` headers) is not done yet; until it is, web storage is unverified and the early warning this ADR relies on is partial.
