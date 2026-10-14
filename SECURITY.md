# Reporting a security issue

Write to cloud@sixta.ai. Say which release you are running (the `vX.Y.Z`
tag, or the output of `sixta-backend --version`), what you found, and how
to reproduce it. Do not open an issue: this repository has no issue
tracker, and a report in a public place reaches everyone before it reaches
us.

We acknowledge every report and keep the reporter informed until it is
settled. A fix ships as a new release here; assets are never replaced in
place, so a fixed version always carries a new tag.

The signing key for the packages in these releases is `RPM-GPG-KEY-sixta`,
attached to every release, and its fingerprint is printed in the README.
Compare that fingerprint with the one Sixta gave you through another
channel before you import the key.
