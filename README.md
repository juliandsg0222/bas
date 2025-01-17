# BAS - Business Automation System

Many organizations and individuals rely on automation across various contexts in their daily operations. With BAS, we aim to provide an open-source platform that empowers users to create customized automation systems tailored to their unique requirements. BAS consists of a series of abstract components designed to facilitate the creation of diverse use cases, regardless of context.

The underlying idea is to develop generic components that can serve a wide range of needs, this approach ensures that all members of the community can leverage the platform's evolving suite of components and use cases to their advantage.

![Gem Version](https://img.shields.io/gem/v/bas?style=for-the-badge)
![Gem Total Downloads](https://img.shields.io/gem/dt/bas?style=for-the-badge)
![Build Badge](https://img.shields.io/github/actions/workflow/status/kommitters/bas/ci.yml?style=for-the-badge)
[![Coverage Status](https://img.shields.io/coveralls/github/kommitters/bas?style=for-the-badge)](https://coveralls.io/github/kommitters/bas?branch=main)
![GitHub License](https://img.shields.io/github/license/kommitters/bas?style=for-the-badge)
[![OpenSSF Scorecard](https://img.shields.io/ossf-scorecard/github.com/kommitters/bas?label=openssf%20scorecard&style=for-the-badge)](https://api.securityscorecards.dev/projects/github.com/kommitters/bas)
[![OpenSSF Best Practices](https://img.shields.io/cii/summary/8713?label=openssf%20best%20practices&style=for-the-badge)](https://bestpractices.coreinfrastructure.org/projects/8713)

## Installation

Install the gem and add to the application's Gemfile by executing:

    $ bundle add bas

If bundler is not being used to manage dependencies, install the gem by executing:

    $ gem install bas

## Requirements

* Ruby 2.6.0 or higher

## Building my own use case

The gem provides with basic interfaces, types, and methods to shape your own use cases in an easy way.

There are 7 currently implemented use cases:
* Birthday notifications - from Notion to Discord
* Next Week Birthday notifications - from Notion to Discord
* PTO notifications - from Notion to Discord
* Next Week PTO notifications - from Notion to Discord
* PTO notifications - from Postgres to Slack
* WIP limit exceeded - from Notion to Discord
* Support email notification - from IMAP to Discord

For this example we'll analyze the birthday notification use case, bringing data from a notion database, and sending the
notifications to a Discord channel.

A *Use Case* object, consists on 5 main components, having it's own responsibility:

### 1. Read - Obtaining the data

Specifically, a reader is an object in charged of bringing data from a data source. The gem already provides the base interface
for building your own reader for your specific data source, or rely on already built classes if they match your purpose.

The base interface for a reader can be found under the `bas/read/base.rb` class. Since this is a implementation of the `Read::Base`
for bringing data from a Notion database, it was created on a new namespace for that data source, it can be found under
`/bas/read/notion/use_case/birthday_today.rb`. It implements specific logic for reading the data and validating the response.

### 2. Serialize - Shaping it

The **Serializer** responsibility, is to shape the data using custom types from the app domain, bringing it into a
common structure understandable for other components, specifically the **Formatter**.

Because of the use case, the Serializer implementation for it, relies on specific types for representing a Birthday it self. It can be found
under `/bas/serialize/notion/birthday_today.rb`

### 3. Formatter - Preparing the message

The **Formatter**, is in charge of preparing the message to be sent in our notification, and give it the right format with the right data.
The template or 'format' to be used should be included in the use case configurations, which we'll review in a further step. It's
implementation can be found under `/bas/formatter/birthday.rb`.

### 4. Process - Optional Data Process

Finally, the **Process** basically, allow required data process depending on the use case like sending formatted messages into a destination. In this case, since the use case was implemented for
Discord, it implements specific logic to communicate with a Discord channel using a webhook. The webhook configuration and name for the 'Sender'
in the channel should be provided with the initial use case configurations. It can be found under `/bas/process/discord/implementation.rb`

### 5. Write - Apply changes in a destination
Finally, the **Write** is in charge of creating or updating information in a destination. This is the last step in the pipeline, and it aims to allow the users to apply changes in a destination depending on the use case. These changes can be a transaction in a database, adding files in a cloud storage, or simply creating logs. For the Birthday use case, after the Process sends the notification, the Write creates a Log in the `$stdout`. It can be found under `lib/bas/write/logs/use_case/console_log.rb`

## Examples

In this example, we demonstrate how to instantiate a birthday notification use case and execute it in a basic Ruby project. We'll also cover its deployment in a serverless configuration, specifically using a simple Lambda deployment.

### Preparing the configurations

We'll need some configurations for this specific use case:
* A **Notion database ID**, from a database with the following structure:

| Complete Name (text) |    BD_this_year (formula)   |         BD (date)        |
| -------------------- | --------------------------- | ------------------------ |
|       John Doe       |       January 24, 2024      |      January 24, 2000    |
|       Jane Doe       |       June 20, 2024         |      June 20, 2000       |

With the following formula for the **BD_this_year** column: `dateAdd(prop("BD"), year(now()) - year(prop("BD")), "years")`

* A Notion secret, which can be obtained, by creating an integration here: `https://developers.notion.com/`, browsing on the **View my integrations** option, and selecting the **New Integration** or **Create new integration** buttons.

* A webhook key, which can be generated directly on discord on the desired channel, following this instructions: `https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks`

* A filter, to determine which data to bring from the database, for this specific case, the filter we used is:

```
#### file.rb ####
today = Date.now
{
    "filter": {
      "or": [
        {
          "property": "BD_this_year",
          "date": {
            "equals": today
          }
        }
      ]
    },
    "sorts": []
}
```

* A template for the formatter to be used for formatting the payload to be send to Discord. For this specific case, the format of the messages should be:

`"NAME, Wishing you a very happy birthday! Enjoy your special day! :birthday: :gift:"`

### Instantiating the Use Case

To instantiate the use case, the `UseCases` module should provide a method for this purpose, for this specific case, the `notify_birthday_from_notion_to_discord`
is the desired one.

**Normal ruby code**
```
filter = {
    "filter": {
      "or": [
        {
          "property": "BD_this_year",
          "date": {
            "equals": today
          }
        }
      ]
    },
    "sorts": []
}

options = {
    read_options: {
      base_url: "https://api.notion.com",
      database_id: NOTION_DATABASE_ID,
      secret: NOTION_API_INTEGRATION_SECRET,
      filter: filter
    },
    process_options: {
      webhook: "https://discord.com/api/webhooks/1199213527672565760/KmpoIzBet9xYG16oFh8W1RWHbpIqT7UtTBRrhfLcvWZdNiVZCTM-gpil2Qoy4eYEgpdf",
      name: "Birthday Bot"
    }
}

use_case = UseCases.notify_birthday_from_notion_to_discord(options)

use_case.perform

```

### Serverless
We'll explain how to configure and deploy a use case with serverless, this example will cover the PTO's notifications use case.

#### Configuring environment variables
Create the environment variables configuration file.

```bash
cp env.yml.example env.yml
```

And put the following env variables
```
dev:
  NOTION_DATABASE_ID: NOTION_DATABASE_ID
  NOTION_SECRET: NOTION_SECRET
  DISCORD_WEBHOOK: DISCORD_WEBHOOK
  DISCORD_BOT_NAME: DISCORD_BOT_NAME
prod:
  NOTION_DATABASE_ID: NOTION_DATABASE_ID
  NOTION_SECRET: NOTION_SECRET
  DISCORD_WEBHOOK: DISCORD_WEBHOOK
  DISCORD_BOT_NAME: DISCORD_BOT_NAME

```

The variables should be defined either in the custom settings section within the `serverless.yml` file to ensure accessibility by all lambdas, or in the environment configuration option for each lambda respectively. For example:

```bash
# Accessible by all the lambdas
custom:
  settings:
      api:
        NOTION_DATABASE_ID: ${file(./env.yml):${env:STAGE}.NOTION_DATABASE_ID}
        NOTION_SECRET: ${file(./env.yml):${env:STAGE}.NOTION_SECRET}}

# Accessible by the lambda
functions:
  lambdaName:
    environment:
      NOTION_DATABASE_ID: ${file(./env.yml):${env:STAGE}.NOTION_DATABASE_ID}
      NOTION_SECRET: ${file(./env.yml):${env:STAGE}.NOTION_SECRET}}
```

#### Schedule
the schedule is configured using an environment variable containing the cron configuration. For example:
```bash
# env.yml file
SCHEDULER: cron(0 13 ? * MON-FRI *)

# serverless.yml
functions:
  lambdaName:
    events:
      - schedule:
        rate: ${file(./env.yml):${env:STAGE}.SCHEDULER}
```

To learn how to modify the cron configuration follow this guide: [Schedule expressions using rate or cron](https://docs.aws.amazon.com/lambda/latest/dg/services-cloudwatchevents-expressions.html)

#### Building your lambda
On your serverless configuration, create your lambda function, on your serverless `/src` folder.

```ruby
# frozen_string_literal: true

require 'bas'

# Initialize the environment variables
NOTION_BASE_URL = 'https://api.notion.com'
NOTION_DATABASE_ID = ENV.fetch('PTO_NOTION_DATABASE_ID')
NOTION_SECRET = ENV.fetch('PTO_NOTION_SECRET')
DISCORD_WEBHOOK = ENV.fetch('PTO_DISCORD_WEBHOOK')
DISCORD_BOT_NAME = ENV.fetch('PTO_DISCORD_BOT_NAME')

module Notifier
  # Service description
  class UseCaseName
    def self.notify(*)
      options = { read_options:, process_options: }

      begin
        use_case = UseCases.use_case_build_function(options)

        use_case.perform
      rescue StandardError => e
        { body: { message: e.message } }
      end
    end

    def self.read_options
      {
        base_url: NOTION_BASE_URL,
        database_id: NOTION_DATABASE_ID,
        secret: NOTION_SECRET,
        filter: {
          filter: { "and": read_filter }
        }
      }
    end

    def self.read_filter
      today = Common::TimeZone.set_colombia_time_zone.strftime('%F').to_s

      [
        { property: 'Desde?', date: { on_or_before: today } },
        { property: 'Hasta?', date: { on_or_after: today } }
      ]
    end

    def self.process_options
      {
        webhook: DISCORD_WEBHOOK,
        name: DISCORD_BOT_NAME
      }
    end

    def self.format_options
      {
        template: ':beach: individual_name is on PTO'
      }
    end
  end
end
```

#### Configure the lambda
In the `serverless.yml` file, add a new instance in the `functions` block with this structure:

```bash
functions:
  ptoNotification:
    handler: src/lambdas/pto_notification.Notifier::PTO.notify
    environment:
      PTO_NOTION_DATABASE_ID: ${file(./env.yml):${env:STAGE}.PTO_NOTION_DATABASE_ID}
      PTO_NOTION_SECRET: ${file(./env.yml):${env:STAGE}.PTO_NOTION_SECRET}
      PTO_DISCORD_WEBHOOK: ${file(./env.yml):${env:STAGE}.PTO_DISCORD_WEBHOOK}
      PTO_DISCORD_BOT_NAME: ${file(./env.yml):${env:STAGE}.DISCORD_BOT_NAME}
    events:
      - schedule:
          name: pto-daily-notification
          description: "Notify every 24 hours at 8:30 a.m (UTC-5) from monday to friday"
          rate: cron(${file(./env.yml):${env:STAGE}.PTO_SCHEDULER})
          enabled: true
```

#### Deploying

Configure the AWS keys:

```bash
serverless config credentials --provider aws --key YOUR_KEY --secret YOUR_SECRET
```

Deploy the project:
```bash
STAGE=prod sls deploy --verbose
```

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and the created tag, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Changelog

Features and bug fixes are listed in the [CHANGELOG][changelog] file.

## Code of conduct

We welcome everyone to contribute. Make sure you have read the [CODE_OF_CONDUCT][coc] before.

## Contributing

For information on how to contribute, please refer to our [CONTRIBUTING][contributing] guide.

## License

The gem is licensed under an MIT license. See [LICENSE][license] for details.

<br/>

<hr/>

[<img src="https://github.com/kommitters/chaincerts-smart-contracts/assets/1649973/d60d775f-166b-4968-89b6-8be847993f8c" width="80px" alt="kommit"/>](https://kommit.co)

<sub>

[Website][kommit-website] •
[Github][kommit-github] •
[X][kommit-x] •
[LinkedIn][kommit-linkedin]

</sub>

[license]: https://github.com/kommitters/bas/blob/main/LICENSE
[coc]: https://github.com/kommitters/bas/blob/main/CODE_OF_CONDUCT.md
[changelog]: https://github.com/kommitters/bas/blob/main/CHANGELOG.md
[contributing]: https://github.com/kommitters/bas/blob/main/CONTRIBUTING.md
[kommit-website]: https://kommit.co
[kommit-github]: https://github.com/kommitters
[kommit-x]: https://twitter.com/kommitco
[kommit-linkedin]: https://www.linkedin.com/company/kommit-co
