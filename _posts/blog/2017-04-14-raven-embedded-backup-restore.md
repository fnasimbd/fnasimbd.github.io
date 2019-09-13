---
layout: post
title: "Backup and Restore RavenDB Embedded Database"
modified: 
categories: blog
excerpt: "How to backup and restore a RavenDB embedded database."
tags: ["C#", ".NET", "RavenDB", "Database"]
image:
feature:
share: true
date: 2017-04-14T00:10:00-00:00
comments: true
---

Recently I used RavenDB embedded database in an application. As part of that, had to take care of its backup and restore operations. RavenDB's official documentation of course covers embedded database backup and restore. I found it, however, bit scattered, not as thorough as the server one's, and some conventions would better be made explicit. As a result, it took me some time to construct the steps from the documentation.

Besides the references in RavenDB official documentation site, a few walkthrough like article would make user's life much easier; I hope there will be a lot of them someday. In this post, I have put together a brief overview of backup and restore, how to do that in RavenDB 3.5, and a few related points.

## Overview of RavenDB's Backup and Restore

RavenDB provides two methods for backup: *full* and *incremental*. Their meaning is as usual: full backup stores the *entire snapshot of a database at some instant*, incremental backup, on the other hand, stores the *difference between the current snapshot of a database and a previous backup*. Incremental backup is naturally faster than full backup, hence preferred for periodic backups.

Backup is an *online* operation: that is you can start a backup while the database is running and processing requests. Restore, on the other hand, is an *offline* operation: you cannot perform a restore without stopping the database.

Backup data (both full and incremental) is stored in a location specified while initiating a backup. A backup location can contain only one full backup; any attempt to perform a full backup to a location that already contains another backup fails with error. An incremental bakcup location, on the other hand, keeps accumulating backup data in the same location on each invocation of incremental backup.

During a restore, user specifies both the backup location and the target restore location. The restore location must not contain another database with the same name. If restoration is succeful the database is available at the restore location in the same state as it was during the time of backup.

Backup of an embedded database must be performed with the RavenDB client API; that is the same API---the `IDocumentStore` and `IDocumentSession` interfaces being part of it---that does other common tasks like creating a database connection, loading and storing data, etc. Restoration of embedded database, however, can be done with both the client API and the *Raven.Server.exe* program.

## Initiating a Backup

Both backup and restore are invoked from an `EmbeddableDocumentStore` instance as administrative [database commands](https://ravendb.net/docs/article-page/3.5/csharp/client-api/commands/what-are-commands).

Initiating a backup of an embedded database is exceptionally simple. Initialize an `EmbeddableDocumentStore` at the database location, call the `StartBackup()` command method with appropritate parameters, the backup location is created if it doesn't exist, and backup starts asynchronously. The `EmbeddableDocumentStore` instance for backup doesn't need to be dedicated for it; you can use the one that is being used by other operations.

{% highlight csharp linenos %}
public void Backup(string backupPath)
{
    try
    {
        DocumentStore.DatabaseCommands.
            GlobalAdmin.StartBackup(backupPath, 
                new DatabaseDocument
                {
                    Id = Database
                },
                false, 
                Database
            );
    }
    catch (Exception ex)
    {
        Logger.Error(ex);
        throw;
    }
}
{% endhighlight %}

Parameters to the `StartBackup` method are obvious except the `databaseDocument`. The `databaseDocument` parameter takes some advanced settings of the database being backed up: like specifying the `Id` in it lets you omit the database name during restore; Raven assumes that from the backup directory.

As mentioned before, backup is an asynchronous operation. The `StartBackup` method returns an `Operation` object that helps tracking completion of backup operation with methods `WaitForCompletion` and its asynchronous counterpart `WaitForCompletionAsync`.

#### Incremental Backup

Before initiating an incremental backup, check the following configurations of your storage engine (RavenDB supports two storage engines: *Esent* and *Voron*; the default is Esent; you can change it to Voron from configuration.) These configurations are mandatory, incremental backup fails if they are not set appropriately.

- **Esent**. Turn off (it is turned on by default) Esent's circular logging during incremental backups. You can do it by setting *Raven/Esent/CircularLog* to `false`.
- **Voron**. Set *Raven/Voron/AllowIncrementalBackups* to `true`.

There is no difference between starting a full backup and an incremental one except the `incremental` parameter; to start an incremental backup, just set the `incremental` boolean parameter to the `StartBackup` method to `true`.

Incremental backup expects a previous *full backup* of the database in the backup location. The very first incremental backup of a database is treated as a full backup by default; no need to set the `incremental` parameter to `false` for first backup. All subsequent incremental backups after the first one are stored as diffs---as timestamped subdirectories inside the backup location.

## Restoring a Database

As mentioned in the overview section, restoration can be done with both the client API and the Raven.Server.exe program. Here I will cover only the client API. The Raven.Server.exe method is very similar to the client one (e.g. the same constraints, same parameters are required, etc.)

Like backup, initiating a restore operation from client API is straightforward. Create and initialize a temporary `EmbeddableDocumentStore` for restore, call `StartRestore` on it, the restore location is created if it doesn't exist, and restore continues asynchronously.

{% highlight csharp linenos %}
public void Restore(string backupPath, string databaseLocation, string databaseName)
{
    using (var documentStore = new EmbeddableDocumentStore())
    {
        documentStore.Initialize();

        documentStore.DatabaseCommands.
            GlobalAdmin.StartRestore(new DatabaseRestoreRequest
            {
                BackupLocation = backupPath,
                DatabaseLocation = databaseLocation,
                DatabaseName = databaseName
            });
    }
}
{% endhighlight %}

Use of the parameter `DatabaseRestoreRequest` passed to `StartRestore` should be obvious. Like backup, restore is also an asynchronous operation. Like `StartBackup`, `StartRestore` also returns an `Operation` object with similar facilities for tracking the operation progress.

If restore is successful, the restored database should be available at the location (`DatabaseLocation`) specified during restore.

#### *EmbeddableDocumentStore* for Restore

Do not set the document store's `DefaultDatabase` property to the name of the database to be restored. Doing so essentially creates a database with that name. Hence restore detects that another database with the same name is online and restoration fails.

#### Backup Location After a Restore from an Incremental Backup

Once you have restored from an incremental backup directory, you can no longer perform incremental backups to that same directory; Raven throws an `InvalidOperationException` with message like "*... Database missed a previous full bakcup before incremental backup.*"

## References

1. [*Administration : Backup and Restore.*](https://ravendb.net/docs/article-page/3.5/csharp/server/administration/backup-and-restore) Backup and restore administration guideline.
2. [*Commands : How to start backup or restore operations?*](https://ravendb.net/docs/article-page/3.5/csharp/client-api/commands/how-to/start-backup-restore-operations) Backup and Resore commands reference.
3. [*RavenDB Configurations Summary.*](https://ravendb.net/docs/article-page/3.5/csharp/server/configuration/configuration-options) Lists all RavenDB configurations.

{% if page.comments %}
<div id="disqus_thread"></div>
<script type="text/javascript">
    /* * * CONFIGURATION VARIABLES * * */
    var disqus_shortname = 'fnasim';

    /* * * DON'T EDIT BELOW THIS LINE * * */
    (function() {
        var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
        dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
        (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
    })();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript" rel="nofollow">comments powered by Disqus.</a></noscript>

{% endif %}

{% include google-analytics.html %}

