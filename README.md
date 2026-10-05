<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PostgreSQL Backup & Replication Lab</title>

    <style>
        * {
            box-sizing: border-box;
        }

        body {
            margin: 0;
            font-family: Arial, sans-serif;
            background: #f4f6f8;
            color: #222;
        }

        header {
            background: #222;
            color: white;
            padding: 30px 20px;
            text-align: center;
        }

        header h1 {
            margin: 0 0 10px;
        }

        .container {
            max-width: 1100px;
            margin: 30px auto;
            padding: 0 15px;
        }

        .step {
            background: white;
            margin-bottom: 25px;
            padding: 25px;
            border-radius: 10px;
            box-shadow: 0 3px 10px rgba(0,0,0,0.1);
        }

        .step h2 {
            margin-top: 0;
            color: #333;
        }

        .description {
            line-height: 1.6;
        }

        pre {
            background: #1e1e1e;
            color: #f5f5f5;
            padding: 18px;
            border-radius: 7px;
            overflow-x: auto;
            line-height: 1.5;
        }

        code {
            font-family: Consolas, monospace;
        }

        .copy-btn {
            background: #007bff;
            color: white;
            border: none;
            padding: 10px 16px;
            border-radius: 5px;
            cursor: pointer;
            margin-top: 8px;
        }

        .copy-btn:hover {
            background: #0056b3;
        }

        .success {
            display: none;
            color: green;
            margin-left: 10px;
        }

        footer {
            text-align: center;
            padding: 25px;
            color: #666;
        }

        @media (max-width: 600px) {
            header h1 {
                font-size: 24px;
            }

            .step {
                padding: 18px;
            }

            pre {
                font-size: 12px;
            }
        }
    </style>
</head>

<body>

<header>
    <h1>Hands-On Lab</h1>
    <p>Backups, Point-in-Time Recovery, and Replication</p>
</header>

<div class="container">

    <!-- STEP 1 -->
    <section class="step">
        <h2>Step 1: Take and Verify a Logical Backup</h2>

        <p class="description">
            Back up the <strong>bootcamp</strong> database and confirm
            that it can be restored successfully.
        </p>

        <pre><code id="code1">mkdir -p ~/backups

pg_dump -Fc -f ~/backups/bootcamp.dump bootcamp

pg_restore --list ~/backups/bootcamp.dump | head

createdb bootcamp_check

pg_restore -d bootcamp_check ~/backups/bootcamp.dump</code></pre>

        <button class="copy-btn" onclick="copyCode('code1', this)">
            Copy Code
        </button>
        <span class="success">Copied!</span>
    </section>


    <!-- STEP 2 -->
    <section class="step">
        <h2>Step 2: Enable WAL Archiving</h2>

        <p class="description">
            Configure PostgreSQL to archive WAL files and take a base backup.
        </p>

        <pre><code id="code2">wal_level = replica
archive_mode = on
archive_command = 'cp %p /home/$USER/backups/wal/%f'</code></pre>

        <button class="copy-btn" onclick="copyCode('code2', this)">
            Copy Code
        </button>
        <span class="success">Copied!</span>

        <h3>Terminal Commands</h3>

        <pre><code id="code3">mkdir -p ~/backups/wal

sudo systemctl restart postgresql

pg_basebackup -D ~/backups/base -Ft -z -Xs -P</code></pre>

        <button class="copy-btn" onclick="copyCode('code3', this)">
            Copy Code
        </button>
        <span class="success">Copied!</span>
    </section>


    <!-- STEP 3 -->
    <section class="step">
        <h2>Step 3: Simulate a Disaster and Recover</h2>

        <p class="description">
            Record the current time, simulate accidental deletion,
            and recover the database to a previous point in time.
        </p>

        <pre><code id="code4">SELECT now();

DELETE FROM students;</code></pre>

        <button class="copy-btn" onclick="copyCode('code4', this)">
            Copy Code
        </button>
        <span class="success">Copied!</span>

        <h3>Recovery Configuration</h3>

        <pre><code id="code5">restore_command = 'cp ~/backups/wal/%f %p'

recovery_target_time = '2025-06-01 10:00:00'</code></pre>

        <button class="copy-btn" onclick="copyCode('code5', this)">
            Copy Code
        </button>
        <span class="success">Copied!</span>

        <h3>Verify Recovery</h3>

        <pre><code id="code6">SELECT count(*) FROM students;</code></pre>

        <button class="copy-btn" onclick="copyCode('code6', this)">
            Copy Code
        </button>
        <span class="success">Copied!</span>
    </section>


    <!-- STEP 4 -->
    <section class="step">
        <h2>Step 4: Set Up a Streaming Standby</h2>

        <p class="description">
            Create a replication role and build a standby server
            using <strong>pg_basebackup</strong>.
        </p>

        <h3>Replication Role</h3>

        <pre><code id="code7">CREATE ROLE replicator
WITH REPLICATION
LOGIN
PASSWORD 'reppass';</code></pre>

        <button class="copy-btn" onclick="copyCode('code7', this)">
            Copy Code
        </button>
        <span class="success">Copied!</span>

        <h3>pg_hba.conf</h3>

        <pre><code id="code8">host replication replicator 127.0.0.1/32 md5</code></pre>

        <button class="copy-btn" onclick="copyCode('code8', this)">
            Copy Code
        </button>
        <span class="success">Copied!</span>

        <h3>Build the Standby</h3>

        <pre><code id="code9">pg_basebackup -h 127.0.0.1 -U replicator -D ~/standby -R -P</code></pre>

        <button class="copy-btn" onclick="copyCode('code9', this)">
            Copy Code
        </button>
        <span class="success">Copied!</span>
    </section>


    <!-- STEP 5 -->
    <section class="step">
        <h2>Step 5: Watch Replication Health</h2>

        <p class="description">
            Observe the replication status and replication lag
            from the primary server.
        </p>

        <pre><code id="code10">SELECT
    application_name,
    state,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;</code></pre>

        <button class="copy-btn" onclick="copyCode('code10', this)">
            Copy Code
        </button>
        <span class="success">Copied!</span>
    </section>


    <!-- WRAP UP -->
    <section class="step">
        <h2>Wrap-Up</h2>

        <p class="description">
            This lab demonstrates how PostgreSQL can be protected using
            logical backups, WAL archiving, Point-in-Time Recovery (PITR),
            and streaming replication.
        </p>

        <ul>
            <li>Logical database backups</li>
            <li>WAL archiving</li>
            <li>Point-in-Time Recovery</li>
            <li>Streaming replication</li>
            <li>Replication health monitoring</li>
        </ul>
    </section>

</div>

<footer>
    PostgreSQL Database Systems – Hands-On Lab
</footer>


<script>
    function copyCode(id, button) {

        const code = document.getElementById(id).innerText;
        const message = button.nextElementSibling;

        navigator.clipboard.writeText(code).then(function () {

            message.style.display = "inline";

            setTimeout(function () {
                message.style.display = "none";
            }, 1500);

        }).catch(function () {
            alert("Unable to copy the code.");
        });
    }
</script>

</body>
</html>
