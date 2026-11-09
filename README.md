# Reproducing Android LITTLE-Core CPU Time Utilization

This README describes the two steps required to reproduce the
experiment: collecting a system-wide Perfetto trace from an Android
phone with eight background applications, and calculating the time-based
utilization of the LITTLE CPU cluster from that trace.

## Step 1. Collect a Perfetto Trace

### Experimental workload and setup

Use a **physical Android phone** connected to a Windows PC over USB.

Before recording, make sure that USB debugging is enabled and that the
phone has authorized the connected PC. The PC should have:

-   Android Platform Tools installed, with `adb` available in
    PowerShell.
-   Python 3 installed and available through the Windows `py` launcher.

The background workload consists of the following eight applications:

1.  Facebook
2.  Instagram
3.  YouTube
4.  Tiktok
5.  ChatGPT
6.  Gemini
7.  X
8.  Gmail

Launch the eight applications and leave them running in the background
during trace collection. Do not force-stop them. Keep the workload
unchanged throughout the recording.

### Recording command

Open PowerShell in the directory where you want to keep the recording
helper script. The following commands download the official Perfetto
Android recording helper, check the ADB connection, and collect a
five-minute trace:

``` powershell
# Download the official Perfetto Android trace-recording helper.
Invoke-WebRequest `
  -Uri "https://raw.githubusercontent.com/google/perfetto/main/tools/record_android_trace" `
  -OutFile ".\record_android_trace.py"

# Check that the Android phone is connected and authorized.
adb devices

# Record a 5-minute system-wide trace.
py -3 .\record_android_trace.py `
  -o "$HOME\Desktop\oppo_cpu_trace.perfetto-trace" `
  -t 5m `
  -b 64mb `
  sched freq idle
```

Make sure that `adb devices` lists the phone with the state `device`
(rather than `unauthorized`).

The recording options are:

-   `-o`: output trace-file path.
-   `-t 5m`: trace duration of five minutes.
-   `-b 64mb`: 64 MB trace buffer.
-   `sched`: records CPU scheduling activity, including which thread
    runs on each CPU and for how long.
-   `freq`: records CPU frequency changes.
-   `idle`: records CPU idle-state events.

When recording finishes, the trace is saved as:

``` text
%USERPROFILE%\Desktop\oppo_cpu_trace.perfetto-trace
```

Open the trace in the [Perfetto UI](https://ui.perfetto.dev) if it is
not opened automatically by the recording helper.

## Step 2. Calculate LITTLE-Core Time Utilization

### Metric definition

For the phone used in this experiment, the CPU topology is:

-   **LITTLE cluster:** CPU 0--CPU 5 (six cores).
-   **Big cluster:** CPU 6--CPU 7 (two cores).

The SQL query below calculates the **time-based CPU utilization** of
each LITTLE core and the average utilization across the six-core LITTLE
cluster.

A row in Perfetto's `sched` table represents a scheduling slice: a
thread running on a CPU for a certain duration. The query uses
`sched.dur` to accumulate the time during which each LITTLE core
executes a non-idle thread.

The calculation follows these definitions:

1. **Per-core busy time:** the sum of the durations of all non-idle
   scheduling slices on that core, clipped to the trace interval. Idle
   execution is excluded by filtering out `utid = 0`, `tid = 0`, and
   thread names matching `swapper%`.

2. **Per-core time utilization:**

   \[ U_i = `\frac{T_{\mathrm{busy},i}}{T_{\mathrm{trace}}}`{=tex}
   `\times 100`{=tex}% \]

3. **Average LITTLE-cluster time utilization:**

   \[ U\_{`\mathrm{LITTLE}`{=tex}} =
   `\frac{\sum_{i=0}^{5} T_{\mathrm{busy},i}}`{=tex} {6
   `\times `{=tex}T\_{`\mathrm{trace}`{=tex}}} `\times 100`{=tex}% \]

The cluster result is normalized by **six cores**. It is therefore the
arithmetic mean of the six per-core utilization values, not the sum of
their utilization percentages. For example, an average utilization of
25% means that the LITTLE cluster has, on average, the equivalent of 1.5
cores busy.

This is a **time-only** metric. It does not weight execution time by CPU
frequency or compute capacity. Although the trace also records frequency
and idle-state events, this SQL query uses scheduling time (`sched`)
only.

The query is system-wide: it counts non-idle threads scheduled on LITTLE
CPUs, including application threads and kernel threads. It is not
restricted to the eight named applications.

> **Portability note:** CPU 0--5 are the LITTLE cores and CPU 6--7 are
> the big cores on the device used for this experiment. CPU numbering
> can differ across devices. If reproducing the experiment on another
> phone, verify its CPU topology and update the `little_cpus` list
> accordingly.

### Perfetto SQL query

Open the collected trace in the Perfetto UI, select the **Query (SQL)**
interface, and execute the following query:

``` sql
WITH
-- 1. Obtain the trace observation interval.
bounds AS (
  SELECT
    start_ts,
    end_ts,
    end_ts - start_ts AS duration_ns
  FROM trace_bounds
),

-- 2. LITTLE cores on the tested device.
little_cpus(cpu) AS (
  VALUES (0), (1), (2), (3), (4), (5)
),

-- 3. Accumulate non-idle running time on each LITTLE core.
--    Scheduling slices are clipped to the trace interval.
per_cpu AS (
  SELECT
    c.cpu,
    b.duration_ns,

    COALESCE(
      SUM(
        CASE
          WHEN s.utid != 0
            AND COALESCE(t.tid, -1) != 0
            AND COALESCE(t.name, '') NOT LIKE 'swapper%'
          THEN
            MIN(s.ts + s.dur, b.end_ts)
            - MAX(s.ts, b.start_ts)
          ELSE 0
        END
      ),
      0
    ) AS busy_ns

  FROM little_cpus c
  CROSS JOIN bounds b
  LEFT JOIN sched s
    ON s.cpu = c.cpu
    AND s.dur > 0
    AND s.ts < b.end_ts
    AND s.ts + s.dur > b.start_ts
  LEFT JOIN thread t
    ON s.utid = t.utid
  GROUP BY
    c.cpu,
    b.duration_ns
),

-- 4. Report per-core utilization and the six-core average.
results AS (
  SELECT
    'CPU' || CAST(cpu AS TEXT) AS cpu_group,
    busy_ns / 1e9 AS busy_time_s,
    100.0 * busy_ns / duration_ns AS utilization_pct,
    0 AS sort_key
  FROM per_cpu

  UNION ALL

  SELECT
    'All Little Cores (Average)' AS cpu_group,
    SUM(busy_ns) / (6.0 * 1e9) AS busy_time_s,
    100.0 * SUM(busy_ns) / (6.0 * MAX(duration_ns))
      AS utilization_pct,
    1 AS sort_key
  FROM per_cpu
)

SELECT
  cpu_group,
  ROUND(busy_time_s, 3) AS busy_time_s,
  ROUND(utilization_pct, 2) AS utilization_pct
FROM results
ORDER BY
  sort_key,
  cpu_group;
```

The result contains one row for each LITTLE core (`CPU0`--`CPU5`) and
one final row (`All Little Cores (Average)`). The columns are:

-   `cpu_group`: the LITTLE core or the six-core average.
-   `busy_time_s`: non-idle running time in seconds. For the average
    row, this is the average busy time per LITTLE core.
-   `utilization_pct`: time utilization as a percentage of the trace
    duration; for the average row, it is normalized by six cores.
