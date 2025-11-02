# Hasura RBAC with Supabase JWT Authentication

In this blog post, we will integrate Hasura RBAC with Supabase JWT Authentication.

## Prerequisites

- Hasura Cloud
- Supabase Cloud


### 1. **Add Custom Claims via Database Function**

Supabase includes custom claims in the JWT from the `raw_app_meta_data` field. You need to set up a database trigger to automatically add Hasura claims when users are created or updated.

Create this function in your Supabase SQL Editor:

```sql
-- Function to add Hasura claims to user metadata
CREATE OR REPLACE FUNCTION public.custom_access_token_hook(event jsonb)
RETURNS jsonb
LANGUAGE plpgsql
AS $$
DECLARE
  claims jsonb;
  user_role text;
BEGIN
  -- Get user role from raw_user_meta_data or default to 'user'
  user_role := COALESCE(
    event->'claims'->'app_metadata'->>'user_role',
    'user'
  );

  -- Build Hasura claims
  claims := jsonb_build_object(
    'https://hasura.io/jwt/claims', jsonb_build_object(
      'x-hasura-user-id', (event->'claims'->>'sub'),
      'x-hasura-default-role', user_role,
      'x-hasura-allowed-roles', jsonb_build_array(user_role, 'anonymous')
    )
  );

  -- Merge with existing claims
  event := jsonb_set(event, '{claims}', (event->'claims') || claims);

  RETURN event;
END;
$$;

-- Grant necessary permissions
GRANT USAGE ON SCHEMA public TO supabase_auth_admin;
GRANT EXECUTE ON FUNCTION public.custom_access_token_hook TO supabase_auth_admin;
REVOKE EXECUTE ON FUNCTION public.custom_access_token_hook FROM authenticated, anon, public;
```


![SQL Editor - Customize Access Token (JWT) Claims hook](media/sql_editor_custom_token_trigger.png)


### 2. **Enable the Hook in Supabase Dashboard**

1. Go to **Authentication** → **Auth Hooks** -> **Add Hook** -> **Customize Access Token (JWT) Claims hook**
![Auth Hooks - Add Customize Access Token (JWT) Claims hook](media/setup_custom_jwt_hook_step1.png)

2. Enable **Customize Access Token (JWT) Claims hook** and select the SQL function: `public.custom_access_token_hook`

![Auth Hooks - Enable Customize Access Token (JWT) Claims hook](media/setup_custom_jwt_hook_step2.png)

3. Save and Verify that the hook is enabled

![Auth Hooks - Verify Customize Access Token (JWT) Claims hook](media/setup_custom_jwt_hook_step3.png)
