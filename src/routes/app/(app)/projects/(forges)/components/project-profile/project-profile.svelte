<script lang="ts" context="module">
  export const PROJECT_PROFILE_FRAGMENT = gql`
    ${PROJECT_PROFILE_HEADER_FRAGMENT}
    ${EDIT_PROJECT_METADATA_FLOW_FRAGMENT}
    ${DRIP_LIST_BADGE_FRAGMENT}
    ${SUPPORT_CARD_PROJECT_FRAGMENT}
    ${EDIT_PROJECT_SPLITS_FLOW_ADDRESS_RECEIVER_FRAGMENT}
    ${EDIT_PROJECT_SPLITS_FLOW_DRIP_LIST_RECEIVER_FRAGMENT}
    ${EDIT_PROJECT_SPLITS_FLOW_PROJECT_RECEIVER_FRAGMENT}
    ${UNCLAIMED_PROJECT_CARD_FRAGMENT}
    ${SPLITS_COMPONENT_PROJECT_SPLITS_FRAGMENT}
    ${SUPPORTERS_SECTION_SUPPORT_ITEM_FRAGMENT}
    fragment ProjectProfile on Project {
      ...UnclaimedProjectCard
      ...ProjectProfileHeader
      ...SupportCardProject
      ...SplitsComponentProjectSplits
      ... on UnclaimedProject {
        account {
          accountId
        }
        support {
          ...SupportersSectionSupportItem
        }
      }
      ... on ClaimedProject {
        ...EditProjectMetadataFlow
        account {
          accountId
        }
        owner {
          accountId
        }
        support {
          ...SupportersSectionSupportItem
        }
      }
    }
  `;
</script>

<script lang="ts">
  import PrimaryColorThemer from '$lib/components/primary-color-themer/primary-color-themer.svelte';
  import SectionHeader from '$lib/components/section-header/section-header.svelte';
  import SplitsIcon from '$lib/components/icons/Splits.svelte';
  import SupportCard, {
    SUPPORT_CARD_PROJECT_FRAGMENT,
  } from '$lib/components/support-card/support-card.svelte';
  import ProjectProfileHeader, {
    PROJECT_PROFILE_HEADER_FRAGMENT,
  } from '$lib/components/project-profile-header/project-profile-header.svelte';
  import UnclaimedProjectCard, {
    UNCLAIMED_PROJECT_CARD_FRAGMENT,
  } from '$lib/components/unclaimed-project-card/unclaimed-project-card.svelte';
  import Wallet from '$lib/components/icons/Wallet.svelte';
  import IdentityBadge from '$lib/components/identity-badge/identity-badge.svelte';
  import SectionSkeleton from '$lib/components/section-skeleton/section-skeleton.svelte';
  import SplitsComponent, {
    SPLITS_COMPONENT_PROJECT_SPLITS_FRAGMENT,
  } from '$lib/components/splits/splits.svelte';
  import ProjectBadge from '$lib/components/project-badge/project-badge.svelte';
  import KeyValuePair from '$lib/components/key-value-pair/key-value-pair.svelte';
  import AggregateFiatEstimate from '$lib/components/aggregate-fiat-estimate/aggregate-fiat-estimate.svelte';
  import Spinner from '$lib/components/spinner/spinner.svelte';
  import Pile from '$lib/components/pile/pile.svelte';
  import ProjectAvatar from '$lib/components/project-avatar/project-avatar.svelte';
  import mapFilterUndefined from '$lib/utils/map-filter-undefined';
  import SupportersSection, {
    SUPPORTERS_SECTION_SUPPORT_ITEM_FRAGMENT,
  } from '$lib/components/supporters-section/supporters.section.svelte';
  import HeadMeta from '$lib/components/head-meta/head-meta.svelte';
  import walletStore from '$lib/stores/wallet/wallet.store';
  import Button from '$lib/components/button/button.svelte';
  import Pen from '$lib/components/icons/Pen.svelte';
  import modal from '$lib/stores/modal';
  import Stepper from '$lib/components/stepper/stepper.svelte';
  import editProjectMetadataSteps, {
    EDIT_PROJECT_METADATA_FLOW_FRAGMENT,
  } from '$lib/flows/edit-project-metadata/edit-project-metadata-steps';
  import AnnotationBox from '$lib/components/annotation-box/annotation-box.svelte';
  import Registered from '$lib/components/icons/Registered.svelte';
  import buildUrl from '$lib/utils/build-url';
  import editProjectSplitsSteps, {
    EDIT_PROJECT_SPLITS_FLOW_ADDRESS_RECEIVER_FRAGMENT,
    EDIT_PROJECT_SPLITS_FLOW_DRIP_LIST_RECEIVER_FRAGMENT,
    EDIT_PROJECT_SPLITS_FLOW_PROJECT_RECEIVER_FRAGMENT,
  } from '$lib/flows/edit-project-splits/edit-project-splits-steps';
  import { fade } from 'svelte/transition';
  import Developer from '$lib/components/developer-section/developer.section.svelte';
  import { goto } from '$app/navigation';
  import { browser } from '$app/environment';
  import isClaimed from '$lib/utils/project/is-claimed';
  import { gql } from 'graphql-request';
  import type {
    ProjectProfileFragment,
    ProjectProfile_ClaimedProject_Fragment,
  } from './__generated__/gql.generated';
  import { DRIP_LIST_BADGE_FRAGMENT } from '$lib/components/drip-list-badge/drip-list-badge.svelte';
  import unreachable from '$lib/utils/unreachable';
  import ShareButton from '$lib/components/share-button/share-button.svelte';
  import highlightStore from '$lib/stores/highlight/highlight.store';
  import breakpointsStore from '$lib/stores/breakpoints/breakpoints.store';
  import dismissablesStore from '$lib/stores/dismissables/dismissables.store';
  import mergeAmounts from '$lib/utils/amounts/merge-amounts';
  import { AddressDriverClient } from 'radicle-drips';
  import DripListAvatar from '$lib/components/drip-list-avatar/drip-list-avatar.svelte';
  import ClaimProjectStepper from '$lib/flows/claim-project-flow/claim-project-stepper.svelte';
  import buildProjectUrl from '$lib/utils/build-project-url';
  import { Forge } from '$lib/graphql/__generated__/base-types';
  import ArrowRight from '$lib/components/icons/ArrowRight.svelte';
  import EyeOpen from '$lib/components/icons/EyeOpen.svelte';

  interface Amount {
    tokenAddress: string;
    amount: bigint;
  }

  export let project: ProjectProfileFragment;

  interface RepoInfo {
    url: string;
    repoName: string;
    ownerName: string;
  }

  export let newRepo: RepoInfo | undefined;
  export let correctCasingRepo: RepoInfo | undefined;

  export let unclaimedFunds: Promise<{ splittable: Amount[]; collectable: Amount[] }> | undefined =
    undefined;
  export let earnedFunds: Promise<Amount[]> | undefined = undefined;

  $: ownAccountId = $walletStore.dripsAccountId;
  $: isOwnProject = ownAccountId === (isClaimed(project) ? project.owner.accountId : undefined);

  function getSplitsPile(
    splitCollections: (
      | ProjectProfile_ClaimedProject_Fragment['splits']['maintainers']
      | ProjectProfile_ClaimedProject_Fragment['splits']['dependencies']
    )[],
  ) {
    const splits = splitCollections.flat();

    return mapFilterUndefined(splits, (v) => {
      switch (v.__typename) {
        case 'AddressReceiver':
          return {
            component: IdentityBadge,
            props: {
              address: v.account.address,
              showIdentity: false,
              outline: true,
              size: 'medium',
            },
          };
        case 'ProjectReceiver':
          return {
            component: ProjectAvatar,
            props: {
              project: v.project,
              outline: true,
            },
          };
        case 'DripListReceiver':
          return {
            component: DripListAvatar,
            props: { outline: true },
          };
        default:
          return undefined;
      }
    });
  }

  function getSupportersPile(supportTable: ProjectProfile_ClaimedProject_Fragment['support'][]) {
    const support = supportTable.flat();

    return mapFilterUndefined(support, (v) => {
      switch (v.__typename) {
        case 'OneTimeDonationSupport':
          return {
            component: IdentityBadge,
            props: {
              address: AddressDriverClient.getUserAddress(v.account.accountId),
              showIdentity: false,
              outline: true,
              size: 'medium',
            },
          };
        case 'ProjectSupport':
          return {
            component: ProjectAvatar,
            props: {
              project: v.project,
              outline: true,
            },
          };
        case 'DripListSupport':
          return {
            component: DripListAvatar,
            props: {
              outline: true,
            },
          };
        default:
          return undefined;
      }
    });
  }

  $: mobileView =
    $breakpointsStore?.breakpoint === 'mobile' || $breakpointsStore?.breakpoint === 'tablet';

  let collectHintTriggered = false;

  function triggerCollectHint() {
    collectHintTriggered = true;

    setTimeout(() => {
      highlightStore.highlight({
        title: 'Collect your earnings',
        description: 'You can collect earnings to your wallet on the Projects screen.',
        element: document.querySelectorAll("[data-highlightid='global-collect']")[0],
        borderRadius: mobileView ? '1rem 0 1rem 1rem' : '2rem 0 2rem 2rem',
        paddingPx: mobileView ? 8 : 0,
      });
    }, 2000);
  }

  const walletInitialized = walletStore.initialized;

  $: {
    if (browser && !collectHintTriggered && $walletInitialized) {
      let url = new URL(window.location.href);

      if (url.searchParams.get('collectHint') === 'true') {
        url.searchParams.delete('collectHint');
        window.history.replaceState({}, '', url.toString());

        if (!dismissablesStore.isDismissed('project-claim-collect-hint')) {
          triggerCollectHint();
          dismissablesStore.dismiss('project-claim-collect-hint');
        }
      }
    }
  }

  $: canonicalRepoInfo = newRepo ?? correctCasingRepo ?? project.source;
</script>

{#if true}
  {@const imageBaseUrl = `/api/share-images/project/${encodeURIComponent(project.source.url)}.png`}
  <HeadMeta
    title="{project.source.ownerName}/{project.source.repoName}"
    description="Support {project.source
      .repoName} on Drips and help make Open-Source Software sustainable."
    image="{imageBaseUrl}?target=og"
    twitterImage="{imageBaseUrl}?target=twitter"
  />
{/if}

<svelte:head>
  <!--
    Canonical URL is either, in order of priority:
    - The new repo URL if the project has been renamed
    - The correct-casing repo URL if the project has different casing to the Drips project
    - The project URL, without ?exact parameter
  -->
  <link
    rel="canonical"
    href="https://drips.network{buildProjectUrl(
      Forge.GitHub,
      canonicalRepoInfo.ownerName,
      canonicalRepoInfo.repoName,
      false,
    )}"
  />
</svelte:head>

<PrimaryColorThemer colorHex={isClaimed(project) ? project.color : undefined}>
  {#if newRepo}
    <div class="notice">
      <AnnotationBox>
        The GitHub repo for this project has been renamed to {newRepo.ownerName}/{newRepo.repoName}.
        <svelte:fragment slot="actions">
          <Button
            icon={ArrowRight}
            variant="primary"
            href={buildProjectUrl(Forge.GitHub, newRepo.ownerName, newRepo.repoName, false)}
            >Go to the new project</Button
          >
        </svelte:fragment>
      </AnnotationBox>
    </div>
  {/if}
  {#if correctCasingRepo}
    <div class="notice">
      <AnnotationBox>
        This project resolves to a GitHub repo with different casing ({correctCasingRepo.ownerName}/{correctCasingRepo.repoName}).
        Any new splits to this misnamed project will automatically be routed to the correct project.
        <svelte:fragment slot="actions">
          <Button
            size="small"
            icon={EyeOpen}
            variant="primary"
            href={buildProjectUrl(
              Forge.GitHub,
              correctCasingRepo.ownerName,
              correctCasingRepo.repoName,
              false,
            )}>View correct project</Button
          >
        </svelte:fragment>
      </AnnotationBox>
    </div>
  {/if}
  {#if !isClaimed(project)}
    <div class="notice">
      <AnnotationBox type="info">
        {#await unclaimedFunds}
          <span />
        {:then res}
          {@const result = res && mergeAmounts(res.splittable, res.collectable)}
          {#if result?.length}This project has <span class="typo-text-small-bold"
              ><AggregateFiatEstimate amounts={result} /></span
            > in claimable funds! Project owners can collect by claiming their project.{:else}This
            project has not been claimed yet but can still receive funds that the owner can collect
            later.{/if}
        {:catch}
          This project is unclaimed.
        {/await}
        <svelte:fragment slot="actions">
          <div class="flex gap-3">
            <ShareButton url={browser ? window.location.href : ''} />
            <Button
              size="small"
              icon={Registered}
              variant="primary"
              on:click={() =>
                $walletStore.connected
                  ? modal.show(ClaimProjectStepper, undefined, {
                      skipWalletConnect: true,
                      projectUrl: project.source.url,
                    })
                  : goto(buildUrl('/app/claim-project', { projectToAdd: project.source.url }))}
              >Claim project</Button
            >
          </div>
        </svelte:fragment>
      </AnnotationBox>
    </div>
  {/if}

  <article class="project-profile" class:claimed={isClaimed(project)}>
    <header class="header">
      {#if isClaimed(project)}
        <div class="owner">
          <span class="typo-text" style:color="var(--color-foreground-level-5)"
            >Project claimed by</span
          >
          <IdentityBadge address={project.owner.address} disableTooltip />
        </div>
      {/if}
      <div>
        <ProjectProfileHeader
          {project}
          editButton={isClaimed(project) && isOwnProject ? 'Edit' : undefined}
          shareButton={{
            url: `https://drips.network${buildProjectUrl(
              Forge.GitHub,
              project.source.ownerName,
              project.source.repoName,
              false,
            )}`,
          }}
          on:editButtonClick={() =>
            isClaimed(project) && modal.show(Stepper, undefined, editProjectMetadataSteps(project))}
        />
      </div>

      {#if isClaimed(project)}
        {#await earnedFunds}
          <div class="flex gap-4">
            <div
              class="stat border rounded-drip-lg h-[6.125rem] w-[11rem] flex items-center justify-center"
            >
              <Spinner />
            </div>
            <div
              class="stat border rounded-drip-lg h-[6.125rem] w-[11rem] flex items-center justify-center"
            >
              <Spinner />
            </div>
          </div>
        {:then earnedFundsResult}
          <div class="stats" in:fade|local={{ duration: 300 }}>
            {#if earnedFundsResult}
              <div class="stat">
                <KeyValuePair key="Donations">
                  <AggregateFiatEstimate amounts={earnedFundsResult} />
                </KeyValuePair>
              </div>
            {/if}
            <!-- ("Supporters" stat) -->
            {#if [project.support].flat().length > 0}
              <a class="stat" href="#support">
                <KeyValuePair key="Supporters">
                  <Pile maxItems={4} components={getSupportersPile([project.support ?? []])} />
                </KeyValuePair>
              </a>
            {/if}
            <!-- ("Splits with" stat) -->
            {#if [project.splits.maintainers, project.splits.dependencies].flat().length > 0}
              <a class="stat" href="#splits">
                <KeyValuePair key="Splits with">
                  <Pile
                    maxItems={4}
                    components={getSplitsPile([
                      project.splits.maintainers ?? [],
                      project.splits.dependencies ?? [],
                    ])}
                  />
                </KeyValuePair>
              </a>
            {/if}
          </div>
        {/await}
      {/if}
    </header>
    <div class="content">
      <Developer accountId={project.account.accountId} />
      {#if isClaimed(project)}
        <section id="splits" class="app-section">
          <SectionHeader
            icon={SplitsIcon}
            label="Splits"
            actions={isOwnProject
              ? [
                  {
                    handler: () =>
                      isClaimed(project) &&
                      modal.show(
                        Stepper,
                        undefined,
                        isClaimed(project)
                          ? editProjectSplitsSteps(project.account.accountId, project.splits)
                          : unreachable(),
                      ),
                    label: 'Edit',
                    icon: Pen,
                  },
                ]
              : []}
          />
          <SectionSkeleton
            loaded={true}
            empty={project.splits.maintainers.length === 0 &&
              project.splits.dependencies.length === 0}
            emptyStateHeadline="No splits"
            emptyStateEmoji="🫧"
            emptyStateText="This project isnʼt sharing incoming funds with any maintainers or dependencies."
          >
            <div class="card">
              <div class="outgoing-splits">
                <ProjectBadge {project} />
                <SplitsComponent
                  list={[
                    {
                      __typename: 'SplitGroup',
                      name: 'Maintainers',
                      list: project.splits.maintainers,
                    },
                    {
                      __typename: 'SplitGroup',
                      name: 'Dependencies',
                      list: project.splits.dependencies,
                    },
                  ]}
                />
              </div>
            </div>
          </SectionSkeleton>
        </section>
      {:else if unclaimedFunds}
        <section class="app-section">
          <SectionHeader icon={Wallet} label="Claimable funds" />
          {#await unclaimedFunds}
            <SectionSkeleton loaded={false} />
          {:then result}
            {@const merged = result && mergeAmounts(result.splittable, result.collectable)}
            <SectionSkeleton loaded={true}>
              <div class="unclaimed-funds-section">
                <UnclaimedProjectCard
                  unclaimedFunds={result}
                  unclaimedTokensExpandable={false}
                  unclaimedTokensExpanded={merged.length > 0}
                  showClaimButton
                  on:claimButtonClick={() =>
                    goto(buildUrl('/app/claim-project', { projectToAdd: project.source.url }))}
                />
              </div>
            </SectionSkeleton>
          {:catch}
            <SectionSkeleton loaded={true} error={true} />
          {/await}
        </section>
      {/if}
      <section id="support">
        <SupportersSection
          accountId={project.account.accountId}
          type="project"
          supportItems={project.support}
        />
      </section>
    </div>
    <aside>
      <div class="become-supporter-card">
        <SupportCard {project} disabled={!!newRepo || !!correctCasingRepo} />
      </div>
    </aside>
  </article>
</PrimaryColorThemer>

<style>
  .project-profile {
    display: grid;
    grid-template-columns: 1fr;
    grid-template-rows: auto auto;
    grid-template-areas:
      'header'
      'content';
    gap: 3rem;
  }

  .project-profile > * {
    min-width: 0;
  }

  .project-profile {
    grid-template-columns: 3fr minmax(auto, 18rem);
    grid-template-rows: auto auto auto;
    grid-template-areas:
      'header sidebar'
      'content sidebar';
  }

  aside {
    grid-area: sidebar;
  }

  .project-profile > * {
    max-width: 100%;
  }

  .content {
    grid-area: content;
    align-self: top;
    display: flex;
    flex-direction: column;
    gap: 3rem;
  }

  .notice {
    margin-bottom: 2rem;
  }

  .header {
    grid-area: header;
    display: flex;
    flex-direction: column;
    gap: 1.5rem;
    margin-bottom: 1rem;
  }

  .header .owner {
    display: flex;
    gap: 0.375rem;
    align-items: center;
  }

  .stats {
    width: calc(100% + 32px);
    margin-left: -16px;
    padding: 0 16px;
    overflow: scroll;
    white-space: nowrap;
  }

  .stats .stat {
    display: inline-flex;
    padding: 1rem;
    border: 1px solid var(--color-foreground);
    border-radius: 1rem 0 1rem 1rem;
    min-height: 6.125rem;
  }
  .stats .stat + .stat {
    margin-left: 0.5rem;
  }

  .become-supporter-card {
    position: sticky;
    top: 6rem;
  }

  .card {
    border: 1px solid var(--color-foreground);
    border-radius: 1rem 0 1rem 1rem;
    overflow: hidden;
  }

  .outgoing-splits {
    padding: 1.5rem;
  }

  .unclaimed-funds-section {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  @media (max-width: 1080px) {
    .project-profile {
      grid-template-columns: minmax(0, 1fr);
      grid-template-rows: auto auto auto;
      gap: 3rem;
      grid-template-areas:
        'header'
        'sidebar'
        'content';
    }

    .header {
      margin-bottom: 0;
    }

    aside {
      height: auto;
    }
  }
</style>
