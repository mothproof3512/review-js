# First push (on your own machine, where the SSH key lives)

1. Create an empty repo on GitHub named `review-js` (no README/license — they are in here),
   or with the GitHub CLI after step 2:  gh repo create review-js --public --source=. --remote=origin --push

2.     unzip review-js.zip && cd review-js
       git init -b main
       git config user.email "mothproof3512@users.noreply.github.com"
       git add .
       git commit -m "review.js 3.0: passive client-side security review from the console"

3.     git remote add origin git@github.com:mothproof3512/review-js.git
       git push -u origin main

Check SSH first if unsure:  ssh -T git@github.com
