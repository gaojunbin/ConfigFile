I'll leave my workaround to enable register for future reference:
- Start the containers as always
  > I'm assuming you're using overleaf toolkit
- Connect to sharelatex container, we need to copy the following files to `/var/lib/sharelatex` so we can edit them
in the host and bind the new version.
  ```bash
  $ cp app/views/user/register.pug /var/lib/sharelatex
  $ app/src/router.js /var/lib/sharelatex
  $ app/src/Features/User/UserController.js /var/lib/sharelatex
  $ app/src/Features/User/UserPagesController.js /var/lib/sharelatex
  ```
- Edit file `data/sharelatex/register.pug` in host (probably you need super user privileges, so do it with `sudo`). Replace
the text inside with this:
  ```pug
  extends ../layout-marketing

  block content
    main.content.content-alt#main-content
      .container
        .row
          .col-md-8.col-md-offset-2.col-lg-6.col-lg-offset-3
            .card
              .page-header
                h1 Register
              if message
                .alert.alert-success= message
              if err_message
                .alert.alert-danger= err_message
              form(action="/register", method="post")
                input(type="hidden", name="_csrf", value=_csrf)
                if email
                  div.form-group
                    label(for="email") Email
                    input#email.form-control(type="email", name="email", value=email, required)
                else
                  div.form-group
                    label(for="email") Email
                    input#email.form-control(type="email", name="email", placeholder="email@example.com", required)
                div.form-group
                  label(for="invitationCode") Invitation Code
                  input#invitationCode.form-control(type="text", name="invitationCode", required)
                button.btn.btn-primary(type="submit") Register
  ```
- Edit the file `data/sharelatex/router.js` and add this line after 
  `webRouter.get('/register', UserPagesController.registerPage)` (around line ~279):
  ```js
      webRouter.post('/register', UserController.register)
  ```
- Edit the file `data/sharelatex/UserController.js` and add these lines at the top:
  ```js
  const UserRegistrationHandler = require('./UserRegistrationHandler')
  const { db, waitForDb } = require('../../infrastructure/mongodb')
  ```
  Add these lines after `clearSessions: expressify(clearSessions),` (around line ~212):
  ```js
  register(req, res, next) {
    const { email, invitationCode } = req.body;
    const csrfToken = req.csrfToken()
    if (invitationCode !== "u530ON5M42jIsFK8QtwMRN1SoBX41X") {
        res.render('user/register', {
          err_message: 'Error: Invalid invitation code.',
          email: email,
          _csrf: csrfToken,
        });
    }
    UserGetter.getUserByAnyEmail(email, { _id: 1 }, (error, user) => {
      if (error) {
        res.render('user/register', {
          err_message: 'Error: ' + JSON.stringify(error),
          _csrf: csrfToken,
        });
      }

      if (!user || !user._id.toString()) {
        UserRegistrationHandler.registerNewUserAndSendActivationEmail(
          email,
          (error, user, setNewPasswordUrl) => {
            if (error) {
              res.render('user/register', {
                err_message: 'Error: ' + JSON.stringify(error),
                _csrf: csrfToken,
              });
            }
            db.users.updateOne(
              { _id: user._id },
              { $set: { isAdmin: false } },
              error => {
                if (error) {
                  res.render('user/register', {
                    err_message: 'Error: ' + JSON.stringify(error),
                    _csrf: csrfToken,
                  });
                }
                res.render('user/register', {
                  message: 'Account created successfully. A confirmation email has been sent.',
                  _csrf: csrfToken,
                });
              }
            );
          }
        );
      } else {
        // Don't reveil if the user already exists in database
        res.render('user/register', {
          message: 'Account created successfully. A confirmation email has been sent.',
          _csrf: csrfToken,
        });
      }
    });
  },
  ```
- Edit the file `data/sharelatex/UserPagesController.js` and add this line after `newTemplateData,` (around line 126):
  ```js
        _csrf: req.csrfToken(),
  ```
- Now edit the file `vim lib/docker-compose.base.yml` and add this lines in `volumes` section:
  ```docker-compose
  - ../data/sharelatex/register.pug:/overleaf/services/web/app/views/user/register.pug:ro
  - ../data/sharelatex/router.js:/overleaf/services/web/app/src/router.js:ro
  - ../data/sharelatex/UserController.js:/overleaf/services/web/app/src/Features/User/UserController.js:ro
  - ../data/sharelatex/UserPagesController.js:/overleaf/services/web/app/src/Features/User/UserPagesController.js:ro
  ```
- Finally, down and up the network:
  ```bash
  bin/docker-compose down && bin/docker-compose up -d
  ```
> **Note**: I've added an invitation code to restrict user registration in my server. You can remove that check if you want (register.pug and UserController.js). Also, don't forget to change the invitation code if you leave it.