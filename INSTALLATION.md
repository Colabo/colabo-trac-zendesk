You will need:
1) this script
2) Zendesk account
3) Trac installation
4) email account
5) specific Zendesk configuration

Installation process:
1. Install ORIGINAL email2trac per its instructions. You can find a sample email2trac.conf that works in sample directory. Ignore "filter_reply" and "filter_reply_dir", you don't need them yet.
2. Install fetchmail and configure it per its instructions. A sample working fetchmail.conf is provided.

3. Verify that it's working. I.e. you can send email to your email account and receive them as tickets in your Trac. AND get replies with "RE:" in subject appear as comments to the ticket.
Go no further until it's working.
Fetchmail launch line looks like "sudo -u apache -i /usr/bin/fetchmail -v --fetchmailrc /var/www/trac/fetchmail.conf --pidfile /tmp/fetchmail.email2trac.pid".

4. Add custom field named "zendesk_ticket" to Trac. A sample part of trac.ini is provided.

5. Proceed to your Zendesk account. We well need to create a few triggers and ticket fields.
In our case, we need to assign the ticket in Trac to some person based on a Zendesk ticket field. If you have other requirements, your experience may vary.
The user story: a supporter receives a ticket. He wants to copy it into Trac. He sets "Trac" field to "Trac John". The ticket get copied into Trac and assigned to user "John". All further comments will also be copied into Trac.

First, add custom field to tickets (Manage > ticket fields):
- Name: Trac
- Type: Dropdown
- Field options: Title "Trac John", Tag "trac-john"; "Trac Pete" + "trac-pete", etc.

Second, add your email to Zendesk as target, we'll need 2 (Settings > extensions > targets):
- Title "trac create", email "my@email.com", subject "{{ticket.title}} ZD{{ticket.id}} #?zendesk_ticket={{ticket.id}}&priority=high"
- Title "trac update", email "my@email.com", subject "RE: {{ticket.title}} ZD{{ticket.id}}"

Next, to triggers (Manage > Triggers and mail notifications):

- Title "trac create"
  -- conditions: "Ticket is created" and "trac" (our custom field) is not "-"
  -- actions:
    --- Notify target "trac create". Message (not including "<<<",">>>"):
<<<
@reporter: {{current_user.email}}
{% if ticket.tags contains 'trac-john' %}
@owner : john@domain.com
{% endif %}
{% if ticket.tags contains 'trac-pete' %}
@owner : pete@domain.com
{% endif %}
'''URL:''' https://{{ticket.url}}
'''Type:''' {{ticket.ticket_type}}
'''Assignee:''' {{ticket.assignee.name}}
'''Requester:''' {{ticket.requester.name}}
'''Tags:''' {{ticket.tags}}

'''Description:'''
{% for comment in ticket.comments limit:1 offset:0 %}
{{comment.value}}
{% for attachment in comment.attachments %}
[{{attachment.url}}  {{attachment.filename}}]
{% endfor %}
{% endfor %}
--endmessage
>>>
    --- Add tags "trac"


- Title "trac create old" (This one is for when we want to copy an old ticket and all of its comments to Trac)
  -- Conditions:
    --- Ticket is updated
    --- Tags contain none of the following: trac
    --- "trac" is not "-"
  -- Actions:
    --- Notify target "trac create". Message (not including "<<<",">>>"):
<<<
@reporter: {{current_user.email}}
{% if ticket.tags contains 'trac-john' %}
@owner : john@domain.com
{% endif %}
{% if ticket.tags contains 'trac-pete' %}
@owner : pete@domain.com
{% endif %}
'''URL:''' https://{{ticket.url}}
'''Type:''' {{ticket.ticket_type}}
'''Assignee:''' {{ticket.assignee.name}}
'''Requester:''' {{ticket.requester.name}}
'''Tags:''' {{ticket.tags}}

'''Description:'''
{% for comment in ticket.comments reversed %}
{% if forloop.first %}
{{comment.value}}
{% for attachment in comment.attachments %}
[{{attachment.url}}  {{attachment.filename}}]
{% endfor %}
{% endif %}
{% endfor %}

'''Comments:'''
{% for comment in ticket.comments reversed %}

{% if forloop.first == false %}
'''{{comment.author.name}}, {{comment.created_at}}:'''
{{comment.value}}
{% for attachment in comment.attachments %}
[{{attachment.url}}  {{attachment.filename}}]
{% endfor %}
{% endif %}
{% endfor %}

--endmessage
>>>
    --- Add tags "trac"

- Title "trac update"
  -- Conditions:
    --- Ticket is updated
    --- Tags contain at least one of the following: trac
    --- Comment is present (public or private)
    --- Status is not closed
  -- Actions:
    --- Notify target "trac update". Message:
<<<
{% for comment in ticket.comments limit:1 offset:0 %}
'''{{comment.author.name}}, {{comment.created_at}}:'''
{{comment.value}}
{% for attachment in comment.attachments %}
[{{attachment.url}}  {{attachment.filename}}]
{% endfor %}
{% endfor %}
--endmessage
>>>


- Title "trac ticket type change"
  -- Conditions:
    --- Ticket type changed
    --- Tags contain at least one of the following: trac
    --- Ticket is updated
    --- Status is not closed
  -- Actions:
    --- Notify target "trac update". Message:
<<<
Zendesk ticket [https://{{ticket.url}} {{ticket.id}}] type changed to {{ticket.ticket_type}}
--endmessage
>>>

- Title "trac ticket assignee change"
  -- Conditions:
    --- Assignee changed
    --- Tags contain at least one of the following: trac
    --- Ticket is updated
    --- Status is not closed
  -- Actions:
    --- Notify target "trac update". Message:
<<<
Zendesk ticket [https://{{ticket.url}} {{ticket.id}}] type changed to {{ticket.assignee}}
--endmessage
>>>


- Title "trac ticket priority change"
  -- Conditions:
    --- Priority changed
    --- Tags contain at least one of the following: trac
    --- Ticket is updated
    --- Status is not closed
  -- Actions:
    --- Notify target "trac update". Message:
<<<
Zendesk ticket [https://{{ticket.url}} {{ticket.id}}] type changed to {{ticket.priority}}
--endmessage
>>>


- Title "trac ticket status change"
  -- Conditions:
    --- Status changed
    --- Tags contain at least one of the following: trac
    --- Ticket is updated
    --- Status is not closed
  -- Actions:
    --- Notify target "trac update". Message:
<<<
Zendesk ticket [https://{{ticket.url}} {{ticket.id}}] type changed to {{ticket.status}}
--endmessage
>>>


- Title "trac ticket requester change"
  -- Conditions:
    --- Requester changed
    --- Tags contain at least one of the following: trac
    --- Ticket is updated
    --- Status is not closed
  -- Actions:
    --- Notify target "trac update". Message:
<<<
Zendesk ticket [https://{{ticket.url}} {{ticket.id}}] type changed to {{ticket.requester}}
--endmessage
>>>


6. Now, replace email2trac with the version from repository. Uncomment "filter_reply" and "filter_reply_dir" options in email2trac.conf. Create "filter_reply_dir".
Run fetchmail. Debug. Repeat until it's working.

