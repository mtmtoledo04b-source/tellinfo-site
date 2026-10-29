name: Deploy via yml

on:
  workflow_dispatch:

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Validate deployment configuration
        env:
          SFTP_HOST: ${{ secrets.SFTP_HOST }}
          SFTP_USER: ${{ secrets.SFTP_USER }}
          SFTP_PASSWORD: ${{ secrets.SFTP_PASSWORD }}
          SFTP_FINGERPRINT: ${{ secrets.SFTP_FINGERPRINT }}
          SFTP_REMOTE_DIR: ${{ secrets.SFTP_REMOTE_DIR }}
          SFTP_BACKUP_DIR: ${{ secrets.SFTP_BACKUP_DIR }}
        run: |
          set -eu
          test -n "$SFTP_HOST" || { echo "Missing SFTP_HOST secret"; exit 1; }
          test -n "$SFTP_USER" || { echo "Missing SFTP_USER secret"; exit 1; }
          test -n "$SFTP_PASSWORD" || { echo "Missing SFTP_PASSWORD secret"; exit 1; }
          test -n "$SFTP_FINGERPRINT" || { echo "Missing SFTP_FINGERPRINT secret"; exit 1; }
          test -n "$SFTP_REMOTE_DIR" || { echo "Missing SFTP_REMOTE_DIR secret"; exit 1; }
          test -n "$SFTP_BACKUP_DIR" || { echo "Missing SFTP_BACKUP_DIR secret"; exit 1; }
          case "$SFTP_REMOTE_DIR" in /*) ;; *) echo "SFTP_REMOTE_DIR must be absolute"; exit 1 ;; esac
          case "$SFTP_BACKUP_DIR" in /*) ;; *) echo "SFTP_BACKUP_DIR must be absolute"; exit 1 ;; esac

      - name: Prepare site files
        run: |
          mkdir -p dist
          cp index.html dist/
          cp -R assets dist/

      - name: Back up current site
        id: backup
        uses: appleboy/ssh-action@v1
        env:
          TARGET_DIR: ${{ secrets.SFTP_REMOTE_DIR }}
          BACKUP_DIR: ${{ secrets.SFTP_BACKUP_DIR }}
          RUN_ID: ${{ github.run_id }}
        with:
          host: ${{ secrets.SFTP_HOST }}
          username: ${{ secrets.SFTP_USER }}
          password: ${{ secrets.SFTP_PASSWORD }}
          fingerprint: ${{ secrets.SFTP_FINGERPRINT }}
          port: 22
          envs: TARGET_DIR,BACKUP_DIR,RUN_ID
          script: |
            set -eu
            : "${TARGET_DIR:?Missing SFTP_REMOTE_DIR secret}"
            : "${BACKUP_DIR:?Missing SFTP_BACKUP_DIR secret}"
            target_dir="${TARGET_DIR%/}"
            backup_dir="${BACKUP_DIR%/}"
            test "$target_dir" = "/home/tellinfo1" || { echo "Unexpected public directory"; exit 1; }
            case "$backup_dir/" in
              "$target_dir/"*) echo "Backup directory must be outside the public directory"; exit 1 ;;
            esac
            case "$target_dir/" in
              "$backup_dir/"*) echo "Backup directory must not contain the public directory"; exit 1 ;;
            esac
            test -d "$target_dir"
            mkdir -p "$backup_dir"
            backup_path="$backup_dir/tellinfo-$RUN_ID"
            mkdir "$backup_path"
            cp -a "$target_dir/." "$backup_path/"

      - name: Remove old site after backup
        uses: appleboy/ssh-action@v1
        env:
          TARGET_DIR: ${{ secrets.SFTP_REMOTE_DIR }}
          BACKUP_DIR: ${{ secrets.SFTP_BACKUP_DIR }}
          RUN_ID: ${{ github.run_id }}
        with:
          host: ${{ secrets.SFTP_HOST }}
          username: ${{ secrets.SFTP_USER }}
          password: ${{ secrets.SFTP_PASSWORD }}
          fingerprint: ${{ secrets.SFTP_FINGERPRINT }}
          port: 22
          envs: TARGET_DIR,BACKUP_DIR,RUN_ID
          script: |
            set -eu
            target_dir="${TARGET_DIR%/}"
            backup_path="${BACKUP_DIR%/}/tellinfo-$RUN_ID"
            test "$target_dir" = "/home/tellinfo1" || { echo "Unexpected public directory"; exit 1; }
            test -d "$backup_path" || { echo "Backup is missing; refusing to remove the current site"; exit 1; }
            find "$target_dir" -mindepth 1 -maxdepth 1 -exec rm -rf -- {} +

      - name: Upload site via SSH
        id: upload
        uses: appleboy/scp-action@v1
        with:
          host: ${{ secrets.SFTP_HOST }}
          username: ${{ secrets.SFTP_USER }}
          password: ${{ secrets.SFTP_PASSWORD }}
          fingerprint: ${{ secrets.SFTP_FINGERPRINT }}
          port: 22
          source: "dist/*"
          target: ${{ secrets.SFTP_REMOTE_DIR }}
          strip_components: 1

      - name: Verify published site
        run: |
          set -euo pipefail
          curl --fail --silent --show-error --location --max-time 30 https://www.tellinfo.com.br/ |
            grep -Fq '<title>TellInfo | Tecnologia que impulsiona o seu negócio</title>'

      - name: Restore backup if deployment fails
        if: failure() && steps.backup.outcome == 'success'
        uses: appleboy/ssh-action@v1
        env:
          TARGET_DIR: ${{ secrets.SFTP_REMOTE_DIR }}
          BACKUP_DIR: ${{ secrets.SFTP_BACKUP_DIR }}
          RUN_ID: ${{ github.run_id }}
        with:
          host: ${{ secrets.SFTP_HOST }}
          username: ${{ secrets.SFTP_USER }}
          password: ${{ secrets.SFTP_PASSWORD }}
          fingerprint: ${{ secrets.SFTP_FINGERPRINT }}
          port: 22
          envs: TARGET_DIR,BACKUP_DIR,RUN_ID
          script: |
            set -eu
            target_dir="${TARGET_DIR%/}"
            backup_path="${BACKUP_DIR%/}/tellinfo-$RUN_ID"
            test "$target_dir" = "/home/tellinfo1" || { echo "Unexpected public directory"; exit 1; }
            test -d "$backup_path" || { echo "Backup is missing; refusing to restore"; exit 1; }
            find "$target_dir" -mindepth 1 -maxdepth 1 -exec rm -rf -- {} +
            cp -a "$backup_path/." "$target_dir/"
