# Rails Database Schema Consistency: A Developer's Guide
aka, Oh no why is schema.rb conflicting all the time!?


## How Database Schemas Diverge

>  "You do it to yourself, you do" -- Radiohead

- Multiple developers create migrations in parallel branches
- Column orders differ due to migrations being run in different sequences
- Database restoration from backups followed by migrations
- Migrations aren't rolled back before switching branches
- Out-of-band database changes without corresponding migrations

## Resolving Schema Differences

1. Use `rails db:prepare` as your primary command for database setup ([guide](https://guides.rubyonrails.org/active_record_migrations.html#preparing-the-database))
   - Handles both empty and partially migrated states
   - Runs schema:load for new databases
   - Runs db:seed

2. For specific scenarios:
   - New database: `rails schema:load` (faster than running all migrations) ([guide](https://guides.rubyonrails.org/active_record_migrations.html#what-are-schema-files-for-questionmark))
   - Existing database: `rails db:migrate` ([guide](https://guides.rubyonrails.org/active_record_migrations.html#running-migrations))
   - Complete reset: `rails db:reset` (drops, creates, and seeds) ([guide](https://guides.rubyonrails.org/active_record_migrations.html#resetting-the-database))
3. When in doubt the only thing that should be considered the source of truth is the production `db:schema:dump`. Production is reality.

## Prevention Strategies

1. Development Practices:
   - Roll back migrations before switching branches[^1]
   - Roll back migrations before deleting migration files[^1]
   - Write idempotent seed files
   - Consider using environment-specific seeds:
     ```ruby
     # seeds.rb
     load(Rails.root.join('db', "#{Rails.env.downcase}_seeds.rb"))
     ```

2. Tools:
   - Consider using `fix-db-schema-conflicts` gem to maintain consistent column ordering
   - Newer versions of Rails includes automatic column sorting in schema dumps ([PR](https://github.com/rails/rails/pull/53281))
   
3. CI/Testing:
   - Consider running `db:reset` in CI before tests
   - Maintaine your seeds.rb file to ensure that it continues to run for developers running (`db:reset` or `db:prepare`). I encourage you to write it so that it is idempotent and can be run again on an existing database as well.

[^1]: Take special care to reset/rollback migrations that you are abandoning, or when switching branches if you are using the`annotate` gem or similar which will modify potentially many additional files.
