# Nightscout Cosmos DB Profile Loader Fix — Local Handoff

Date: 2026-06-02

Repository:
- GitHub fork: https://github.com/Metatron135/cgm-remote-monitor
- Local path: /Users/metatron/dev/nightscout/cgm-remote-monitor
- Upstream/original project: https://github.com/nightscout/cgm-remote-monitor

Azure app:
- App name: zenleo
- URL: https://zenleo.azurewebsites.net
- Current deployment type: Azure App Service running a Docker container image
- Previous image source: Docker Hub official Nightscout image
  - Registry login server: nightscout
  - Image and tag: cgm-remote-monitor:latest
- Patched image source after the fix:
  - Registry login server: ghcr.io
  - Image and tag currently tested successfully:
    ghcr.io/metatron135/cgm-remote-monitor:fd65724949166e7d5c5d299c31676fe87cc24c95

Important context:
- The Nightscout installation uses Azure Cosmos DB with the MongoDB API.
- Nightscout officially warns that Cosmos DB / MongoDB compatibility is not perfect.
- The official Azure setup docs mention that the older Cosmos DB video/transcript is obsolete because Cosmos DB is not fully compatible with Nightscout and cannot be deployed in some geographical areas.
- However, for this project Azure + Cosmos DB is currently the only viable free-plan setup, so the goal was not to migrate away, but to patch the compatibility issue.

Problem summary:
- Nightscout was stuck in a loop showing this browser popup:
  “Redirecting you to the Profile Editor to create a new profile.”
- The API endpoint /api/v1/profile.json showed that a valid profile document existed.
- The profile collection in Cosmos DB also contained a valid profile document.
- However, internally Nightscout runtime data was loading zero profiles.
- Because runtime ddata.profiles was empty, the Basal plugin believed there was no treatment profile.
- When ENABLE contained “basal”, the console and server logs repeatedly showed:
  “For the Basal plugin to function you need a treatment profile”
- Disabling “basal” made the popup loop stop, confirming the loop was triggered by Basal plugin failing because no runtime profile was loaded.
- But disabling basal is not a real long-term solution because future pump/loop integrations may need basal/profile data.

Key observations during debugging:
- /api/v1/status.json showed the app was healthy:
  status: ok
  apiEnabled: true
  careportalEnabled: true
- /api/v1/profile.json showed profile documents existed.
- Profile examples included fields like:
  defaultProfile: "Default"
  store.Default.dia
  store.Default.carbratio
  store.Default.sens
  store.Default.timezone
  store.Default.basal
  store.Default.target_low
  store.Default.target_high
  startDate
  mills
  units
- Adding fields such as isValid and identifier did not solve the original runtime loading problem by itself.
- The key symptom was:
  - API-level profile endpoint could return the profile.
  - Runtime ddata loader still did not load the profile through ctx.profile.last().
- Browser console debugging showed cases where:
  - Nightscout.client.data existed.
  - Nightscout.client.data.profiles was [].
  - Nightscout.client.profile runtime object sometimes did not exist or profile.hasData() was false.
  - Nightscout.client.settings.isEnabled('basal') returned true when basal was enabled.
- This confirmed the problem was not browser cache and not the profile editor UI itself.
- The issue was server-side runtime profile loading from Cosmos DB.

Relevant source code:
- File:
  lib/data/dataloader.js
- Original function:
  loadProfile(ddata, ctx, callback)
- Original behavior:
  It called ctx.profile.last(function(err, results) { ... })
  If results were empty or missing, it did not load any profile into ddata.profiles.
- The Basal plugin later depends on ddata.profiles having at least one valid profile.
- Because Cosmos DB did not behave as expected for the existing profile.last() path, Nightscout failed to load the runtime profile even though the API endpoint could show the document.

Original loadProfile() area before patch:
- Around lib/data/dataloader.js lines ~421-438.
- Original logic:
  function loadProfile(ddata, ctx, callback) {
      ctx.profile.last(function(err, results) {
          if (!err && results) {
              var profiles = [];
              results.forEach(function(element) {
                  if (element) {
                      profiles[0] = element;
                  }
              });
              ddata.profiles = profiles;
          }
          callback();
      });
  }

Patch goal:
- Keep normal Nightscout behavior untouched when ctx.profile.last() works.
- Only if ctx.profile.last() returns no profiles, try a Cosmos DB-compatible fallback query directly against the profile collection.
- Query the profile collection directly:
  ctx.store.collection('profile').find({}).sort({ startDate: -1 }).limit(1).toArray()
- Load the fallback result into ddata.profiles.
- Add clear console warnings to confirm whether fallback is being used.

First patch:
- Branch created:
  fix/cosmos-profile-loader
- File modified:
  lib/data/dataloader.js
- First commit:
  34216aa fix: add Cosmos DB fallback for profile loader
- Merged into master:
  a7a90a3 merge: Cosmos DB profile loader fallback

First patch issue:
- The initial fallback used callback-style toArray(function(...)).
- In the deployed Node/MongoDB driver environment, this did not complete correctly.
- Log showed:
  “Profile loader returned no profiles; trying Cosmos DB fallback query”
- But it did not show:
  “Cosmos DB profile fallback loaded profiles: 1”
- This meant the callback-style fallback query did not return as expected.

Second patch:
- Updated fallback to Promise-based toArray().
- Commit:
  fd65724949166e7d5c5d299c31676fe87cc24c95
- Commit message:
  fix: use promise-based profile fallback query
- This is the commit that successfully worked in Azure.

Patched loadProfile behavior:
- It now defines applyProfileResults(results).
- If ctx.profile.last() returns profiles, it uses those.
- If ctx.profile.last() returns empty/no results, it logs:
  “Profile loader returned no profiles; trying Cosmos DB fallback query”
- Then it runs a direct profile collection query:
  ctx.store.collection('profile')
    .find({})
    .sort({ startDate: -1 })
    .limit(1)
    .toArray()
    .then(...)
    .catch(...)
- On success it logs:
  “Cosmos DB profile fallback loaded profiles: 1”
- Then it calls applyProfileResults(fallbackResults).
- The runtime data loader then shows:
  profiles:1

Current expected patched loadProfile function shape:
- It should include this logic:

function loadProfile(ddata, ctx, callback) {
    function applyProfileResults(results) {
        var profiles = [];

        if (results && results.length) {
            results.forEach(function(element) {
                if (element) {
                    profiles[0] = element;
                }
            });
        }

        ddata.profiles = profiles;
        callback();
    }

    ctx.profile.last(function(err, results) {
        if (!err && results && results.length) {
            return applyProfileResults(results);
        }

        console.warn('Profile loader returned no profiles; trying Cosmos DB fallback query');

        try {
            ctx.store.collection('profile')
                .find({})
                .sort({ startDate: -1 })
                .limit(1)
                .toArray()
                .then(function(fallbackResults) {
                    console.warn('Cosmos DB profile fallback loaded profiles:', fallbackResults ? fallbackResults.length: 0);
                    return applyProfileResults(fallbackResults);
                })
                .catch(function(fallbackErr) {
                    console.warn('Cosmos DB profile fallback failed:', fallbackErr);
                    return applyProfileResults(results);
                });
        } catch (fallbackException) {
            console.warn('Cosmos DB profile fallback exception:', fallbackException);
            return applyProfileResults(results);
        }
    });
}

Validation:
- After deploying the patched image from commit fd65724949166e7d5c5d299c31676fe87cc24c95, Azure logs showed:

Profile loader returned no profiles; trying Cosmos DB fallback query
Cosmos DB profile fallback loaded profiles: 1
Load Complete:
     sgvs:784, treatments:1, profiles:1, devicestatus:22
data loaded: reloading sandbox data and updating plugins

- This confirms that:
  1. The normal Nightscout profile loader still fails with Cosmos DB.
  2. The fallback activates.
  3. The fallback successfully loads the profile.
  4. Runtime ddata now contains profiles:1.
  5. The Basal plugin no longer complains about missing treatment profile.
  6. The popup loop stops.

Docker / GHCR work:
- A new GitHub Actions workflow was added:
  .github/workflows/publish-ghcr.yml
- Commit:
  7c1c349 ci: publish patched image to GHCR
- The workflow publishes the patched image to GHCR.
- The successful GHCR image after the second patch was tagged by full commit SHA:
  ghcr.io/metatron135/cgm-remote-monitor:fd65724949166e7d5c5d299c31676fe87cc24c95
- In Azure Deployment Center, the app was changed to:
  Source: Container Registry
  Image source: Other container registries
  Image type: Public
  Registry login server: ghcr.io
  Image and tag: metatron135/cgm-remote-monitor:fd65724949166e7d5c5d299c31676fe87cc24c95
- Startup command left empty.
- After saving and restarting the app, Azure successfully pulled and ran the patched image.

Important GitHub Actions note:
- There are two workflows now:
  1. Original main.yml / “CI test and publish Docker image”
  2. New publish-ghcr.yml / “Publish patched Nightscout image to GHCR”
- The original upstream workflow can fail in matrix tests for one Node/MongoDB combination, but this does not necessarily block the GHCR publish workflow.
- The GHCR publish workflow is the important one for Azure deployment.
- In the successful run, the GHCR publish workflow completed and produced the image.
- Do not assume the original Docker Hub publish job is relevant because it only publishes for github.repository_owner == 'nightscout'. It will not publish Docker Hub images from this fork.

Known issue:
- The current patch is a targeted workaround, not a complete upstream-quality fix.
- The normal ctx.profile.last() path still returns no profiles in Cosmos DB.
- The fallback fixes runtime loading by querying the profile collection directly.
- Logs will still show:
  “Profile loader returned no profiles; trying Cosmos DB fallback query”
  This is expected with Cosmos DB until the underlying ctx.profile.last() incompatibility is fixed.
- Successful behavior is confirmed by:
  “Cosmos DB profile fallback loaded profiles: 1”
  and:
  “Load Complete: ... profiles:1 ...”

Current status:
- The user confirmed: “All working.”
- The patched deployment solved the popup loop.
- Basal can be used again because runtime profile data now loads correctly.
- Important: do not remove the fallback unless the Cosmos DB profile.last() behavior is fixed and verified.

Relevant commits:
- 34216aa fix: add Cosmos DB fallback for profile loader
- a7a90a3 merge: Cosmos DB profile loader fallback
- 7c1c349 ci: publish patched image to GHCR
- fd65724949166e7d5c5d299c31676fe87cc24c95 fix: use promise-based profile fallback query

Commands that were used during the process:
- Checked repository state:
  git remote -v
  git status
  git branch --show-current
  git log -1 --oneline
  node -e "console.log(require('./package.json').version)"
  grep -R "function loadProfile" -n lib/data
  grep -R "Load Complete" -n lib/data
  grep -R "profile.last" -n lib

- Confirmed Nightscout version:
  package.json version was 15.0.7

- Created branch:
  git checkout -b fix/cosmos-profile-loader

- Inspected source:
  sed -n '400,465p' lib/data/dataloader.js

- Verified syntax:
  node -c lib/data/dataloader.js

- Checked direct store collection usage:
  grep -R "store.collection" -n lib | head -20

- Committed first patch:
  git add lib/data/dataloader.js
  git commit -m "fix: add Cosmos DB fallback for profile loader"
  git push -u origin fix/cosmos-profile-loader

- Merged first patch:
  git checkout master
  git merge --no-ff fix/cosmos-profile-loader -m "merge: Cosmos DB profile loader fallback"
  git push origin master

- Added GHCR workflow:
  .github/workflows/publish-ghcr.yml

- Committed GHCR workflow:
  git add .github/workflows/publish-ghcr.yml
  git commit -m "ci: publish patched image to GHCR"
  git push origin master

- Got current commit SHA:
  git rev-parse HEAD

- Second patch commit:
  git commit -m "fix: use promise-based profile fallback query"
  git push origin master

Profile document context:
- The profile document in Cosmos DB has a valid structure similar to:

{
  "_id": ObjectId("6a1e6fb01556593bd64b301f"),
  "isValid": true,
  "identifier": "default-profile",
  "defaultProfile": "Default",
  "store": {
    "Default": {
      "dia": 3,
      "carbratio": [
        {
          "time": "00:00",
          "value": 30,
          "timeAsSeconds": 0
        }
      ],
      "carbs_hr": 20,
      "delay": 20,
      "sens": [
        {
          "time": "00:00",
          "value": 100,
          "timeAsSeconds": 0
        }
      ],
      "timezone": "UTC",
      "basal": [
        {
          "time": "00:00",
          "value": 0.1,
          "timeAsSeconds": 0
        }
      ],
      "target_low": [
        {
          "time": "00:00",
          "value": 3.9,
          "timeAsSeconds": 0
        }
      ],
      "target_high": [
        {
          "time": "00:00",
          "value": 10,
          "timeAsSeconds": 0
        }
      ],
      "startDate": "1970-01-01T00:00:00.000Z",
      "units": "mmol"
    }
  },
  "startDate": "2026-01-02T00:00:00.000Z",
  "mills": 1767193200000,
  "created_at": "2026-06-02T05:58:28.468Z",
  "srvModified": 1780379908468,
  "units": "mmol"
}

Important nuance:
- The profile endpoint showed the profile, and API-level queries could see it.
- But server runtime loader could not load it through ctx.profile.last().
- Therefore, future debugging should distinguish:
  1. API endpoint returns profile.
  2. Runtime ddata.profiles contains profile.
- The second one is what matters for basal/plugin behavior.

What basal is responsible for:
- The basal plugin visualizes or uses basal rate information from the active treatment profile.
- It needs a valid runtime treatment profile containing basal schedule data.
- It is especially relevant for pump/loop-related Nightscout usage because basal rates are part of the therapy profile.
- Disabling basal only hides/turns off the basal plugin behavior; it does not delete profile data.
- The correct fix is not “disable basal forever”, but “make runtime profile loading work”, which is what this patch did.

How to verify in future:
1. In Azure log stream after restart, look for:
   Profile loader returned no profiles; trying Cosmos DB fallback query
   Cosmos DB profile fallback loaded profiles: 1
   Load Complete:
        ... profiles:1 ...
2. Confirm the popup no longer appears:
   “Redirecting you to the Profile Editor to create a new profile.”
3. If basal is enabled, confirm this message does not appear:
   “For the Basal plugin to function you need a treatment profile”
4. Open browser console and optionally check:
   Nightscout.client.settings.isEnabled('basal')
   It should return true if basal is enabled.
5. Runtime data should contain a profile:
   Nightscout.client.data.profiles
   It should not be an empty array.

Known deployment note:
- Azure may show previous container logs mixed with current container logs.
- Lines prefixed with “[Previous Container]” can be from an older instance.
- Always check the latest boot sequence and whether it shows the patched fallback success line.

Rollback plan:
- If needed, Azure can be pointed back to:
  nightscout / cgm-remote-monitor:latest
- But doing so will likely reintroduce the Cosmos DB runtime profile loading problem.
- Safer rollback within patched fork would be to deploy a previous GHCR image tag, but currently the working tag is:
  ghcr.io/metatron135/cgm-remote-monitor:fd65724949166e7d5c5d299c31676fe87cc24c95

Future improvement:
- This patch should eventually be cleaned up into a more upstream-friendly fix if desired.
- Better long-term approach may involve fixing ctx.profile.last() or the underlying storage/query abstraction for Cosmos DB compatibility.
- Current patch is pragmatic and intentionally narrow:
  only profile runtime loading
  only fallback when normal loader returns no profiles
  no changes to profile editor
  no changes to API profile endpoint
  no changes to basal plugin itself
