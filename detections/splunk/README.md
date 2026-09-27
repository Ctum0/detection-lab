# Splunk detections

Splunk SPL searches auto-generated from `../sigma/` by CI
(`.github/workflows/validate.yml`, `{stem}.spl` per rule). Never hand-edit —
change the Sigma source instead. `ssh_success_after_failures.yml` is
excluded (temporal correlation); see the workflow comments.
