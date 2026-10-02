# ARI is the only verse address

Store, routes and database columns address verses by upstream's ARI instead of a (book index, chapter index) pair. Markers, progress marks, plans and sync all key on ARI, so choosing a different address now would force a data migration and break sync compatibility later. Web deep links take the form `/reader?ari=...`.
