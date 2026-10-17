# Imported Version bytes on web live in IndexedDB

The web build keeps imported `.yes` files in IndexedDB, one record per file name, behind the same `VersionFileStore` interface native uses. The user-data rows (which Versions are installed) stay in `expo-sqlite` as ADR 0003 decides.

Considered and rejected: storing the bytes in `expo-sqlite`, because a multi-megabyte BLOB through the alpha web adapter would test the least-proven path with the largest payload. The Origin Private File System was also considered; IndexedDB is supported by every browser the app targets and needs no feature check beyond `indexedDB` existing.

Consequence: clearing site data removes the bytes but not the row. The Version then shows as "file missing" and can be removed, as on native when its file is gone.
