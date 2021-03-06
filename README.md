#Stormpath is Joining Okta
We are incredibly excited to announce that [Stormpath is joining forces with Okta](https://stormpath.com/blog/stormpaths-new-path?utm_source=github&utm_medium=readme&utm-campaign=okta-announcement). Please visit [the Migration FAQs](https://stormpath.com/oktaplusstormpath?utm_source=github&utm_medium=readme&utm-campaign=okta-announcement) for a detailed look at what this means for Stormpath users.

We're available to answer all questions at [support@stormpath.com](mailto:support@stormpath.com).


## Stormpath Organizations Authority Example

The purpose of this example is to show how you can hook into the 
Stormpath Spring Security integration to limit access based on
belonging to a Stormpath Organization

### Stormpath Setup

In this example, you need to create two Stormpath Organizations: one for
Admins and one for Users.

Map at least one Stormpath Directory for each Organization and ensure that
each Directory has at least one account.

Finally, Map the two Organizations to your Stormpath Application.

### Build

```bash
mvn clean package
```

### Run

```bash
STORMPATH_API_KEY_FILE=<path to apiKey.properties> \
STORMPATH_APPLICATION_HREF=<full href to Stormpath Application> \
STORMPATH_AUTHORIZED_ORG_USER=<full href to Stormpath Organization for Admins> \
STORMPATH_AUTHORIZED_ORG_ADMIN=<full href to Stormpath Organization for Users> \
java -jar target/*.jar
```

### Notes

The key to having this work is to override a `Bean` for determining Account
Authorities.

Here's what this looks like:

```java
@Configuration
public class AccountGrantedAuthorityConfig {

    @Bean
    public AccountGrantedAuthorityResolver stormpathAccountGrantedAuthorityResolver() {
        return new AccountGrantedAuthorityResolver() {

            @Override
            public Set<GrantedAuthority> resolveGrantedAuthorities(Account account) {
                Set<GrantedAuthority> set = new HashSet<>();

                OrganizationAccountStoreMappingList mappings =
                    account.getDirectory().getOrganizationAccountStoreMappings();

                for (OrganizationAccountStoreMapping mapping : mappings) {
                    set.add(new SimpleGrantedAuthority(mapping.getOrganization().getHref()));
                }

                return set;
            }
        };
    }
}
```

For a given Stormpath Account, the `Bean` is creating a `Set` of `GrantedAuthority` where each item in the 
set represents the `href` of the Stormpath Organization that Stormpath Account belongs to.

The `OrgService` class has methods that cannot be entered unless the logged in user belongs to a certain
Organization:

```java
@Service
public class OrgService {

    @PreAuthorize("hasAuthority(@orgs.USER)")
    public void assertInUserOrg() {}


    @PreAuthorize("hasAuthority(@orgs.ADMIN)")
    public void assertInAdminOrg() {}
}
```

This takes advantage of the Spring Security `PreAuthorize` annotation with the `hasAuthority` construct.

You may wonder where `@orgs.USER` and `@orgs.ADMIN` is coming from. The `@` tells Spring Security to look
for a `Bean`.

If you look at the `Orgs` class you can see what's going on:

```java
@Component
public class Orgs {
    public final String USER;
    public final String ADMIN;

    @Autowired
    public Orgs(Environment env) {
        USER = env.getProperty("stormpath.authorized.org.user");
        ADMIN = env.getProperty("stormpath.authorized.org.admin");
    }
}
```

The `@Component` annotation ensure that this will be exposed as a `Bean`. By default, the bean name will
be the class name, camel-cased. Thus, `orgs` is the bean name.

Spring automatically converts all-caps and underlined environment variables to the lowercase dotted form.
This way, the environment properties can come from the system environment (in all caps as shown above) or from the 
`application.properties` (as dotted lowercase) file in your project.

Thus, `@orgs.USER` resolves to the `href` of the Stormpath Organization passed in on the command line as a
system environment variable.
