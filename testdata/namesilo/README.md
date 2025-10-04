# Solver testdata directory

This directory contains credentials that are used when running the test suite.
Because the tests actually perform live updates via the Namesilo.com API, you need to supply valid Namesilo.com credentials.

## Namesilo.com username

Copy [`config.json.sample`](config.json.sample) to `config.json` and update the `username` field to use your Namesilo.com username.

## Namesilo.com API token

Copy [`namesilo-credentials.yml.sample`](namesilo-credentials.yml.sample) to `namesilo.credentials` and update the `api-token` field to use your Namesilo.com API token.